# AOG-AgTronik Bridge -- AvMap AgTronic Section Control

Bridge application connecting **AgOpenGPS** to the Agromehanika **AG-Tronik** M1/S1 sprayer controller via the PAVPAGT NMEA-like serial protocol based on reverse engineering the AvMap serial section control.

---

## Overview

```
 AgOpenGPS / Rate Controller          Bridge              AvMap AgTronic
 +--------------------------+    +---------------+    +------------------+
 |  Section control (Auto)  |--->| UDP :8888     |    |                  |
 |  Speed data              |    |               |--->| SCT (sections)   |
 |  Hello heartbeat         |    |  Serial TX    |    | SPD (speed)      |
 |                          |    |  115200 8N1   |    | WDT (probe)      |
 |  PGN 0xEA (sect data)   |<---|               |    | VER (version)    |
 |  PGN 0xED (from machine)|<---| UDP :9999     |    |                  |
 |  PGN32618 (switch box)  |<---|               |<---| SWT (switch sts) |
 |  Hello reply             |<---|  Serial RX    |<---| ACK              |
 +--------------------------+    +---------------+    +------------------+
```

---

## Supported Features

| Feature | Description |
|---------|-------------|
| Bidirectional section control | Sends section ON/OFF commands to AvMap; reports machine state back to AgOpenGPS |
| Auto mode | AgOpenGPS controls sections; disabled sections on AvMap are forced off |
| Manual mode | AvMap controls sections locally; state is reported back to AgOpenGPS |
| Master switch | ON/OFF state reported to AgOpenGPS; controls all sections |
| Speed forwarding | GPS speed from AgOpenGPS sent to AvMap display (speed * 10) |
| Section widths | Automatically read from machine WDT response and saved to config |
| Configurable section count | Supports 5, 7, or 9 section machine variants |
| Configurable rates | SCT and SPD command frequencies adjustable in config |
| Immediate section change | Section commands sent instantly on change (no wait for next cycle) |
| Comms-lost safety | Sections zeroed when AgIO connection is lost |
| Half-duplex timing | 50ms gap between serial commands to prevent response corruption |
| Auto-reconnect | 4-state machine handles connection loss and recovery |

---

## PAVPAGT Serial Protocol

### Physical Layer

- **Baud rate:** 115200
- **Format:** 8N1 (8 data bits, no parity, 1 stop bit)
- **Line ending:** `\r\n`

### Message Format

```
$PAVPAGT,<CMD>[,<arg1>,<arg2>,...]*<CS>\r\n
```

- `$` -- sentence start
- `PAVPAGT` -- talker/sentence ID
- `<CMD>` -- command identifier
- `<args>` -- comma-separated arguments (optional)
- `*` -- checksum delimiter
- `<CS>` -- 2-character uppercase hex XOR checksum

### Checksum Calculation

XOR all ASCII characters between `$` and `*` (exclusive):

```python
def pavpagt_checksum(body: str) -> str:
    x = 0
    for ch in body:
        x ^= ord(ch)
    return f"{x:02X}"
```

Example: `$PAVPAGT,WDT*2E` -- body = `PAVPAGT,WDT`, XOR = `0x2E`

---

## Commands (Bridge -> AvMap)

### WDT -- Watchdog / Connection Probe

```
$PAVPAGT,WDT*2E
```

- Sent at 1 Hz during DISCONNECTED and CONNECTED states only
- Used to detect machine presence
- Machine responds with section widths

### VER -- Version Query

```
$PAVPAGT,VER*25
```

- Sent once after first valid machine response
- Machine responds with firmware version string

### SCT -- Section Control

```
$PAVPAGT,SCT,1,1,0,0,0,1,1*XX
```

- Sends N section states (0=off, 1=on) matching configured section count
- Sent at configurable rate (default 1 Hz) + immediately on change from AgOpenGPS
- Only sent in RUNNING state (machine connected + AgIO connected)

### SPD -- Speed

```
$PAVPAGT,SPD,125*XX
```

- Sends speed as integer value = km/h * 10 (e.g., 125 = 12.5 km/h)
- Sent at configurable rate (default 2 Hz)
- Only sent in RUNNING state

---

## Responses (AvMap -> Bridge)

### SWT -- Switch Status

```
$PAVPAGT,SWT,A,1,0,0,0,0,0,1,1*XX
```

| Field | Description |
|-------|-------------|
| `A` or `M` | Mode: Auto or Manual |
| `1` or `0` | Main/Master switch: ON or OFF |
| `0,0,0,0,0,1,1` | Per-section states (N values matching section count) |

**Interpretation by mode:**

- **Auto mode, section=1:** Section is enabled for auto control by AgOpenGPS
- **Auto mode, section=0:** Section is disabled/forced-off on the machine
- **Manual mode, section=1:** Section is actively spraying
- **Manual mode, section=0:** Section is off

**Keepalive:** The machine periodically sends `$PAVPAGT,SWT*39` (no data fields) as a heartbeat.

### VER -- Version Response

```
$PAVPAGT,VER,1.23*XX
```

- Returns firmware version string

### ACK -- Acknowledgement

```
$PAVPAGT,ACK,SCT*XX
$PAVPAGT,ACK,SPD*XX
```

- Acknowledges received commands (SCT, SPD)

### WDT -- Watchdog Response (Section Widths)

```
$PAVPAGT,WDT,0250,0250,0250,0300,0250,0250,0250*XX
```

- Echoes WDT with section widths in centimeters
- Values are zero-padded 4-digit integers
- Saved to config file automatically

---

## State Machine

```
DISCONNECTED ──[valid response]──> CONNECTED
                                      │
                                   [send VER]
                                      │
                              [VER response]──> READY
                                                  │
                                          [AgIO connects]──> RUNNING
                                                                │
DISCONNECTED <──[3s timeout]── any state                        │
READY <──────[AgIO timeout]──────────────────────────────── RUNNING
```

| State | WDT | VER | SCT | SPD | Meaning |
|-------|-----|-----|-----|-----|---------|
| DISCONNECTED | 1 Hz | -- | -- | -- | Probing for machine |
| CONNECTED | 1 Hz | once | -- | -- | Machine found, querying version |
| READY | -- | -- | -- | -- | Machine ready, waiting for AgIO |
| RUNNING | -- | -- | yes | yes | Full operation |

---

## AgOpenGPS UDP Communication

### Incoming (AgIO -> Bridge) on port 8888

| PGN | Hex | Description | Key Data |
|-----|-----|-------------|----------|
| Hello | 0xC8 | AgIO heartbeat | Triggers Hello reply |
| Machine Data | 0xEF | Section commands from AgOpenGPS | Byte 11: section bitmask 1-8 |
| Steer Data | 0xFE | Speed + section commands | Bytes 5-6: speed (LE, *0.1 km/h), Byte 11: sections |

### Outgoing (Bridge -> AgIO) on port 9999

#### PGN 0xEA (234) -- Section Control Data

The primary response PGN processed by AgOpenGPS and Rate Controller.

```
Byte:  0    1    2    3    4    5    6-8  9    10   11   12   13
       0x80 0x81 0x7B 0xEA 0x08 MAIN RES  ON_L OFF_L ON_H OFF_H CRC
```

| Byte | Field | Description |
|------|-------|-------------|
| 0-1 | Header | `0x80 0x81` |
| 2 | Source | `0x7B` (123 = machine module) |
| 3 | PGN | `0xEA` (234) |
| 4 | Length | `0x08` (8 data bytes) |
| 5 | Main SW bits | bit0=MasterOn (momentary), bit1=MasterOff (momentary) |
| 6-8 | Reserved | `0x00` |
| 9 | Sections ON 1-8 | Bitmask of active sections (Manual mode only) |
| 10 | Sections OFF 1-8 | Bitmask of disabled/forced-off sections (always) |
| 11 | Sections ON 9-16 | Bitmask of active sections (Manual mode only) |
| 12 | Sections OFF 9-16 | Bitmask of disabled/forced-off sections (always) |
| 13 | CRC | Sum of bytes 2-12, masked to 8 bits |

**Mode-dependent behavior:**

| Mode | ON bytes (9, 11) | OFF bytes (10, 12) | Main SW bits (5) |
|------|------------------|--------------------|------------------|
| Auto | `0x00` (AGO controls) | Disabled sections | Momentary on change |
| Manual | Active relay state | Inactive sections | Momentary on change |

**Main SW bits are momentary:** Only set non-zero on the cycle where mode/master changes, then immediately cleared to `0x00`. This prevents continuous re-triggering in AgOpenGPS.

#### PGN 0xED (237) -- From Machine (compatibility)

```
Byte:  0    1    2    3    4    5    6    7-12 13
       0x80 0x81 0x7B 0xED 0x08 RL   RH   RES  CRC
```

| Byte | Field | Description |
|------|-------|-------------|
| 5 | relay_lo | Current section state bitmask 1-8 |
| 6 | relay_hi | Current section state bitmask 9-16 |

#### PGN 0x7B (123) -- Hello Reply

```
Byte:  0    1    2    3    4    5    6    7-9  10
       0x80 0x81 0x7B 0x7B 0x05 RL   RH   RES  CRC
```

- Sent in response to AgIO Hello (0xC8) when machine is READY or RUNNING
- Makes the machine module icon go green in AgOpenGPS

#### PGN 32618 -- Switch Box (Rate Controller)

```
Byte:  0    1    2     3    4    5
       0x6A 0x7F FLAGS SW_L SW_H CRC
```

| Byte | Field | Description |
|------|-------|-------------|
| 0 | HeaderLo | `106` (0x6A) |
| 1 | HeaderHi | `127` (0x7F) |
| 2 | Flags | bit0=Auto, bit1=MasterOn, bit2=MasterOff |
| 3 | sw0-sw7 | Section switch states 1-8 |
| 4 | sw8-sw15 | Section switch states 9-16 |
| 5 | CRC | Sum of bytes 0-4, masked to 8 bits |

- Only sent when mode, master switch, or section states change (not periodic)
- CRC uses simple sum (not the AgOpenGPS 0x80/0x81 preamble style)

---

## Configuration File (`config_pavpagt.ini`)

```ini
[main]
com = COM48
comms_lost_zero = 1
sections = 7
sct_hz = 1
spd_hz = 2
subnet = 255.255.255.255
section_widths = 250,250,250,300,250,250,250
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `com` | `0` (prompt) | Serial port (e.g., `COM3`). Set to `0` to prompt on startup |
| `comms_lost_zero` | `1` | Zero all sections when AgIO connection is lost |
| `sections` | `7` | Number of sections (5, 7, or 9) |
| `sct_hz` | `1` | Section command send rate in Hz |
| `spd_hz` | `2` | Speed command send rate in Hz |
| `subnet` | `255.255.255.255` | UDP broadcast address for AgIO communication |
| `section_widths` | (auto) | Section widths in cm, read from machine WDT response |

---

## Threading Architecture

```
+------------------+     +------------------+     +------------------+
| UDP Listener     |     | Serial Receiver  |     | Periodic TX      |
| (port 8888)      |     | (115200 baud)    |     | (50ms tick)      |
|                  |     |                  |     |                  |
| - Receives PGNs  |     | - Parses PAVPAGT |     | - WDT probe      |
| - Updates target |     | - Handles SWT    |     | - SCT commands   |
|   sections/speed |     | - State changes  |     | - SPD commands   |
| - Sends replies  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
         |                        |                        |
         +------------------------+------------------------+
                                  |
                      +-----------+-----------+
                      | PAVPAGTRequester      |
                      | (shared state + locks)|
                      +-----------------------+
```

All threads are daemon threads. A keyboard thread (press X to exit) provides clean shutdown.

---

## Data Flow Examples

### Auto Mode -- Normal Operation

1. AgOpenGPS sends section mask `0x7F` (sections 1-7 ON) via PGN 0xEF byte 11
2. Bridge sends `$PAVPAGT,SCT,1,1,1,1,1,1,1*XX` to AvMap
3. AvMap responds `$PAVPAGT,SWT,A,1,1,1,1,1,1,1,1*XX` (all enabled)
4. Bridge sends PGN 0xEA with ON=0x00 (auto, AGO controls), OFF=0x00 (none disabled)

### Auto Mode -- Sections Disabled on Machine

1. Operator disables sections 1-5 on AvMap display
2. AvMap responds `$PAVPAGT,SWT,A,1,0,0,0,0,0,1,1*55`
3. Bridge sends PGN 0xEA with ON=0x00, OFF=0x1F (sections 1-5 forced off)
4. AgOpenGPS disables sections 1-5, only 6-7 remain in auto control

### Manual Mode -- Section Control

1. AvMap in Manual mode, operator enables sections 3-4
2. AvMap sends `$PAVPAGT,SWT,M,1,0,0,1,1,0,0,0*XX`
3. Bridge sends PGN 0xEA with ON=0x0C (sections 3-4), OFF=0x73 (1,2,5,6,7)
4. AgOpenGPS displays sections 3-4 as active

### Master Switch Toggle

1. Operator turns off master switch on AvMap
2. AvMap sends `$PAVPAGT,SWT,A,0,0,0,0,0,0,0,0*XX`
3. Bridge sends PGN 0xEA with byte 5 = 0x02 (MasterOff, momentary)
4. Next cycle: byte 5 = 0x00 (cleared)
5. AgOpenGPS registers master off, all sections stop

---

## Known Protocol Behaviors

| Behavior | Handling |
|----------|----------|
| Concatenated messages without `\r\n` | Parser splits on `$` boundaries |
| Double `$$` prefix | Garbage stripping before `$PAVPAGT,` |
| Garbage bytes on power-on | Prefix stripping finds `$PAVPAGT,` within line |
| Empty SWT keepalive (`$PAVPAGT,SWT*39`) | Handled silently at DEBUG level |
| Back-to-back SCT/SPD causes corruption | 50ms gap (REQUEST_GAP_S) between commands |
| WDT response includes section widths | Parsed and saved to config automatically |

---

## Build and Run

### Development

```bat
python AOG_PAVPAGT_bridge.py
```

### Standalone Executable

```bat
build_exe_pavpagt.bat
```

Uses PyInstaller to create a single-file `.exe` with all dependencies bundled.

### Requirements

- Python 3.8+
- `pyserial` package
- Windows (uses `msvcrt` for keyboard input)
