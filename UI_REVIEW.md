# Mobilinkd BLEConfig Android App — UI Review Notes

## Overview

The BLEConfig Android app configures Mobilinkd TNC (Terminal Node Controller) devices
over Bluetooth Low Energy (BLE). It uses a fragment-based navigation architecture with
a central `MainActivity`, a `TncViewModel` for shared state, and a `TncInterface` for
BLE KISS protocol communication.

## Screen/Feature Map

### 1. SelectDeviceFragment
- **Purpose:** Discover and select a paired BLE TNC device
- **Key interactions:**
  - Requests Bluetooth permissions (Android 12+ uses BLUETOOTH_SCAN/CONNECT; older uses BLUETOOTH + location)
  - Requests Bluetooth enable via system dialog
  - Filters scan to only show bonded devices with matching service UUID (`00000001-ba2a-46c9-ae49-01b0961f68bb`)
  - 10-second scan timeout with manual rescan option via toolbar menu
  - Shows spinner during scanning, fades main content to indicate scan activity
  - Tap device → navigate to ConnectingFragment
  - Only bonded (paired) devices are shown

### 2. ConnectingFragment
- **Purpose:** Establish BLE connection to selected device
- **Key interactions:**
  - Initiates BLE connection via MainActivity.connect()
  - Handles connection success → MainMenuFragment
  - Handles connection failure → pops back to SelectDeviceFragment
  - Handles disconnect while active → pops back to SelectDeviceFragment
  - Also handles reconnect scenarios from MainMenuFragment resume

### 3. MainMenuFragment
- **Purpose:** Hub of configuration options after connection
- **Buttons (6):**
  - TNC Information
  - Modem Configuration
  - KISS Parameters
  - Receive Audio
  - Transmit Audio
  - Power Settings
- **Behavior:**
  - On resume, fetches all TNC values via tncInterface.getAllValues()
  - Sets device name in toolbar
  - All 6 buttons always visible; tap navigates to respective fragment

### 4. TncInformationFragment
- **Purpose:** Display device identity and firmware info (read-only)
- **Fields displayed:**
  - Hardware Version
  - Firmware Version
  - MAC Address
  - Serial Number
  - Date/Time
- **Behavior:**
  - Sets TNC datetime from device on resume
  - All fields read-only via LiveData observers

### 5. ModemConfigurationFragment
- **Purpose:** Configure modem type and signal polarity
- **Controls:**
  - Modem Type spinner (populated dynamically from tncSupportedModemTypes)
  - Supported types: 1200 AFSK, 300 AFSK, 9600 FSK, M17
  - Pass All toggle switch
  - Receive Polarity toggle switch
  - Transmit Polarity toggle switch
- **Behavior:**
  - Updates TNC via tncInterface.setModemType(), setPassAll(), setReceivePolarity(), setTransmitPolarity()

### 6. KissParametersFragment
- **Purpose:** Configure KISS protocol timing parameters
- **Controls:**
  - TX Delay slider (0–255, step 1)
  - Persistence slider (0–255, step 1)
  - Slot Time slider (0–255, step 1)
  - Duplex toggle switch
- **Behavior:**
  - Updates via tncInterface.setTxDelay(), setPersistence(), setSlotTime(), setDuplex()
  - Sliders show current value from TNC

### 7. PowerSettingsFragment
- **Purpose:** Display battery level and USB power behavior
- **Controls:**
  - Battery level slider (read-only, color-coded: red <3400mV, yellow <3700mV, green ≥3700mV)
  - Battery level label in mV
  - USB Power On toggle (enable USB power when TNC powers on)
  - USB Power Off toggle (enable USB power when TNC powers off)
- **Behavior:**
  - Polls battery via tncInterface.getBatteryLevel() on resume

### 8. ReceiveAudioFragment
- **Purpose:** Adjust receive input gain and twist, monitor input level
- **Controls:**
  - Input Gain slider
  - Input Twist slider
  - Receive Audio Level bar (live-updating, color-coded by level)
- **Behavior:**
  - Starts receive audio stream on resume via tncInterface.startReceiveAudio()
  - Stops on pause via tncInterface.stopReceiveAudio()
  - Updates via setInputGain(), setInputTwist()

### 9. TransmitAudioFragment
- **Purpose:** Adjust transmit gain/twist, test transmission with tone
- **Controls:**
  - PTT Style chip selector: Simplex (0) / Multiplex (1)
  - Transmit Gain slider (disabled until Transmit pressed)
  - Transmit Twist slider (disabled until Transmit pressed)
  - Transmit button (toggle, enables sliders when active)
  - Test Tone chip selector: Mark / Space / Both
- **Behavior:**
  - PTT Off sent on pause
  - Sends PTT Mark/Space/Both depending on test tone selection
  - Sliders enabled only while Transmit is active

## Architecture Notes

### Navigation
- Jetpack Navigation with NavController
- Fragment-to-fragment navigation via action IDs
- Source tracking to handle back/resume edge cases

### State Management
- TncViewModel shared across all fragments via activityViewModels()
- BLE data flows: BluetoothLEService → KissDecoder → TncViewModel → Fragment bindings
- TncInterface wraps KISS command encoding and BLE output stream

### Key BLE Service UUID
- TNC Service: `00000001-ba2a-46c9-ae49-01b0961f68bb`

### Permission Model
- Android 12+ (API 31+): BLUETOOTH_SCAN, BLUETOOTH_CONNECT
- Older: BLUETOOTH, BLUETOOTH_ADMIN, ACCESS_FINE_LOCATION

### Data Persistence
- Settings changed via TncInterface commands are saved to TNC EEPROM via saveIfChanged()
- Called in MainActivity.onStop()

## Web UI Design Implications

A web implementation using Web Bluetooth API would replicate:
1. **Device Scanning** — navigator.bluetooth.requestDevice() with the TNC service UUID
2. **Connection** — device.gatt.connect() / disconnect()
3. **Characteristic Reads/Writes** — GATT operations for KISS protocol
4. **6 configuration screens** mapped to tabs in a single-page UI
5. **Live audio level** — not directly possible in web (no audio stream from TNC via BLE to web)
6. **Transmit test** — possible with audio output, but complex

The web mockup in `index.html` demonstrates the structural layout matching these fragments.