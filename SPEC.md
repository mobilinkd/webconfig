# Mobilinkd TNC Web Configurator — Requirements Specification

## 1. Project Overview

**Name:** Mobilinkd TNC Web Configurator  
**Type:** Web application (SPA)  
**Core Functionality:** Configure Mobilinkd BLE TNCs via Web Bluetooth, cross-platform (macOS, Windows, Linux)  
**Target Users:** Amateur radio operators using Mobilinkd TNC hardware on desktop/laptop computers where the Android app is not available

## 2. Technology Stack

- **Runtime:** Browser-only (Web Bluetooth API) — no native app, no Electron
- **Framework:** Vanilla JS or lightweight SPA (no heavy framework required for a config tool)
- **Bluetooth:** [Web Bluetooth API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API)
- **Transport:** BLE GATT — KISS protocol over BLE (per [BLE-KISS-API spec](https://github.com/hessu/aprs-specs/blob/master/BLE-KISS-API.md))
- **Build:** Static HTML/CSS/JS, hostable anywhere

## 3. BLE Service & Protocol

### 3.1 GATT Service UUIDs

| Item | UUID |
|------|------|
| KISS Service | `00000001-ba2a-46c9-ae49-01b0961f68bb` |
| TX Characteristic (write, no response) | `00000002-ba2a-46c9-ae49-01b0961f68bb` |
| RX Characteristic (notify) | `00000003-ba2a-46c9-ae49-01b0961f68bb` |

### 3.2 BLE MTU

- TNC fixed MTU: **160 bytes**
- OS handles fragmentation; no application-level fragmentation needed
- Web Bluetooth does not expose MTU negotiation control

### 3.3 KISS Protocol

Standard KISS framing over BLE characteristic reads/writes:
- **FEND:** `0xC0`
- **FESC:** `0xDB` → escaped as `FESC + TFEND (0xDC)` or `FESC + TFESC (0xDD)`
- **Frame format:** `FEND + port/cmd nibble + data... + FEND`
- Multi-frame concatenation supported
- Frames may span multiple BLE transfer units

### 3.4 SetHardware Commands (KISS cmd 0x06)

All commands sent as KISS frames with cmd=0x06 (SetHardware) and device-specific subcommands:

| Subcmd | Name | Args | Description |
|--------|------|------|-------------|
| 0x01 | SET_OUTPUT_GAIN | 2 bytes (16-bit signed) | TX output gain |
| 0x02 | SET_INPUT_GAIN | 2 bytes (16-bit signed) | RX input gain |
| 0x03 | SET_SQUELCH_LEVEL | 1 byte | Squelch threshold |
| 0x05 | STREAM_VOLUME | 0 | Start audio level streaming |
| 0x06 | GET_BATTERY_LEVEL | 0 | Returns battery voltage (mV) |
| 0x07 | PTT_MARK | 0 | PTT on MARK frequency |
| 0x08 | PTT_SPACE | 0 | PTT on SPACE frequency |
| 0x09 | PTT_BOTH | 0 | PTT on both frequencies |
| 0x0A | PTT_OFF | 0 | Release PTT |
| 0x0C | GET_OUTPUT_VOLUME | 0 | Get TX output volume |
| 0x10 | SET_VERBOSITY | 1 byte | Log verbosity level |
| 0x18 | SET_INPUT_TWIST | 1 byte | RX twist setting |
| 0x1A | SET_OUTPUT_TWIST | 1 byte | TX twist setting |
| 0x2A | SAVE_EEPROM | 0 | Persist settings to EEPROM |
| 0x2B | ADJUST_INPUT_LEVELS | 0 | Trigger audio level adjustment |
| 0x31 | GET_DATETIME | 0 | Get TNC date/time (BCD) |
| 0x32 | SET_DATETIME | 8 bytes BCD | Set TNC date/time |
| 0x45 | SET_BT_CONN_TRACK | 1 byte | Bluetooth connection tracking |
| 0x49 | SET_USB_POWER_ON | 1 byte | USB power on behavior |
| 0x4B | SET_USB_POWER_OFF | 1 byte | USB power off behavior |
| 0x4F | SET_PTT_CHANNEL | 1 byte | PTT channel selection |
| 0x50 | GET_PTT_CHANNEL | 0 | Get PTT channel |
| 0x51 | SET_PASSALL | 1 byte | Pass all frames mode |
| 0x53 | SET_RX_POLARITY | 1 byte | RX polarity invert |
| 0x55 | SET_TX_POLARITY | 1 byte | TX polarity invert |
| 0x7F | GET_ALL_VALUES | 0 | Retrieve all config values |

Extended (cmd=0x06, subcmd=-0x3D = 0xC1):
| Subcmd | Name | Args | Description |
|--------|------|------|-------------|
| 0xC1, 0x82 | SET_MODEM_TYPE | 1 byte | Modem type (1=1200 AFSK, 2=300 AFSK, 3=9600 FSK, 5=M17) |
| 0xC1, 0x83 | GET_SUPPORTED_MODEM_TYPES | 0 | Returns bitmap of supported modems |

### 3.5 Standard KISS Parameters

| KISS Cmd | Name | Range | Description |
|----------|------|-------|-------------|
| 0x01 | TX_DELAY | 0–255 | Pre-keyup delay in 10ms units |
| 0x02 | PERSISTENCE | 0–255 | CSMA persistence (p = value/256) |
| 0x03 | SLOT_TIME | 0–255 | Slot interval in 10ms units |
| 0x04 | TX_TAIL | 0–255 | Post-keyup hold time in 10ms units |
| 0x05 | FULL_DUPLEX | 0/1 | Full duplex mode |

## 4. Feature Requirements

### 4.1 Device Discovery & Connection

- Scan for BLE peripherals advertising `KTS_SERVICE_UUID`
- Display device name (from advertising data `Complete Local Name` or GATT device name)
- OS pairing dialog triggered on first connect
- Persist bonded device for auto-reconnect
- Connection state: Idle → Scanning → Connecting → Connected → Error
- Handle dual-mode MAC conflict (same TNC paired over BLE and Classic BT)
- Graceful error handling: device not found, pairing failed, connection lost

### 4.2 TNC Information (Read-Only)

- Firmware version
- Hardware version
- Serial number
- MAC address
- Battery voltage (mV) with visual indicator
- Device name

### 4.3 KISS Parameters

- TX Delay: slider/input (0–255, step 1)
- Persistence: slider/input (0–255, step 1)
- Slot Time: slider/input (0–255, step 1)
- TX Tail: slider/input (0–255, step 1)
- Full Duplex: toggle switch

### 4.4 Modem Configuration

- Modem type selector (from supported types): 1200 AFSK, 300 AFSK, 9600 FSK, M17
- Pass All toggle
- RX Polarity toggle
- TX Polarity toggle

### 4.5 Audio Settings

- RX Input Gain: slider with range from device capabilities
- RX Input Twist: slider with range from device capabilities
- TX Output Gain: slider with range from device capabilities
- TX Output Twist: slider with range from device capabilities
- Squelch Level: slider

### 4.6 Power Settings

- Battery level display (mV with color-coded bar)
- USB Power On toggle
- USB Power Off toggle

### 4.7 PTT Control

- Transmit control: PTT Off / PTT Mark / PTT Space / PTT Both
- PTT Style selector (channel)

### 4.8 Date/Time

- Display current TNC date/time (UTC)
- Set TNC date/time from browser clock

### 4.9 Save/Restore

- Save settings to TNC EEPROM
- Unsaved changes indicator
- Confirm before disconnecting with unsaved changes

## 5. Non-Functional Requirements

### 5.1 Browser Compatibility

- **Supported:** Chrome 56+, Edge 79+, Brave (latest)
- **Not supported:** Firefox (no Web Bluetooth), Safari (broken on most platforms), Internet Explorer
- **Platforms:** macOS, Windows 10/11, Linux (BlueZ)

### 5.2 UX Requirements

- Single-page application, no page reloads
- Responsive layout (works on desktop and tablet)
- Clear connection status indicator
- Loading states for async BLE operations
- Error messages shown inline, not in browser alerts
- No credentials or account required

### 5.3 Offline Support

- Works offline after initial load (no external API calls except Web Bluetooth)
- Static file hosting sufficient

## 6. UI Structure

```
┌─────────────────────────────────────────┐
│  [Logo]  Mobilinkd TNC Config    [?]   │  ← Header
├─────────────────────────────────────────┤
│  ┌─────────┐  ┌──────────────────────┐   │
│  │ Device  │  │                      │   │
│  │ List /  │  │   Settings Panel     │   │
│  │ Status  │  │                      │   │
│  └─────────┘  └──────────────────────┘   │
│                                         │
│  ┌─ TNC Info ────────────────────────┐  │
│  │ Firmware | Hardware | Serial |   │  │
│  │ Battery | MAC Address             │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌─ KISS Parameters ─────────────────┐  │
│  │ TX Delay | Persistence | Slot    │  │
│  │ TX Tail | Full Duplex            │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌─ Modem Configuration ─────────────┐  │
│  │ Modem Type | Pass All | Polarity  │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌─ Audio Settings ──────────────────┐  │
│  │ RX Gain | RX Twist | TX Gain |    │  │
│  │ TX Twist | Squelch               │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌─ Power Settings ──────────────────┐  │
│  │ Battery | USB Power On/Off        │  │
│  └───────────────────────────────────┘  │
│                                         │
│  [ Save to Device ]  [ Revert ]        │
└─────────────────────────────────────────┘
```

## 7. Out of Scope

- Direct KISS data connection (APRS operation) — this is config only
- Firmware update
- Device firmware dump
- Bluetooth Classic / RFCOMM support
- iOS Safari (broken Web Bluetooth support)
- Android (use the native Android app)

## 8. References

- [BLE-KISS-API Specification](https://github.com/hessu/aprs-specs/blob/master/BLE-KISS-API.md)
- [KISS Protocol (Wikipedia)](https://en.wikipedia.org/wiki/KISS_(amateur_radio_protocol))
- [Web Bluetooth API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API)
- [Mobilinkd BLEConfig Android App](https://github.com/mobilinkd/BLEConfig)
