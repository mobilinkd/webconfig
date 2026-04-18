# Mobilinkd TNC BLE Protocol Reference

Based on the Mobilinkd BLEConfig Android app
(`com.mobilinkd.bleconfig`, GitHub `mobilinkd/BLEConfig`).

---

## 1. BLE GATT UUIDs

| Role | UUID | Notes |
|------|------|-------|
| **TNC Service** | `00000001-ba2a-46c9-ae49-01b0961f68bb` | Primary service |
| **TX Characteristic** (write, no response) | `00000002-ba2a-46c9-ae49-01b0961f68bb` | App → TNC commands |
| **RX Characteristic** (notify) | `00000003-ba2a-46c9-ae49-01b0961f68bb` | TNC → app responses |
| **CCCD Descriptor** | `00002902-0000-1000-8000-00805f9b34fb` | Standard client characteristic config |

### Connection Flow

```
1. Device must be BOND_BONDED (paired) before open() will succeed.
2. connect() → device.connectGatt() with TRANSPORT_LE
3. onServicesDiscovered() → getService(TNC_SERVICE_UUID)
   → getCharacteristic(TNC_SERVICE_TX_UUID)
   → getCharacteristic(TNC_SERVICE_RX_UUID)
4. txCharacteristic.writeType = WRITE_TYPE_NO_RESPONSE
5. gatt.requestMtu(517)   ← requested, platform-dependent what actually negotiates
6. onMtuChanged() → mtu = negotiated_mtu − 3   (subtracts 3 for BLE overhead)
7. gatt.setCharacteristicNotification(rxCharacteristic, true)
8. writeDescriptor(CCCD, ENABLE_NOTIFICATION_VALUE)
9. onDescriptorWrite(GATT_SUCCESS) → onConnected()
```

**MTU behavior:** The app requests 517 bytes (API 21+). The actual MTU is negotiated
by the platform; the usable payload per write is `mtu − 3`. The TNC sends responses
via notifications on the RX characteristic. Duplicate FEND bytes at the start of a
frame are silently ignored (decoder "GET_CMD" state ignores them).

**Pairing note:** For dual-mode devices, if not paired via BLE, an authorization or
pairing prompt can appear during step 8 (descriptor write). If the user accepts,
notification is enabled. If declined, the OS closes the connection.
Paired BLE-only devices do not trigger this prompt.

**Retry behavior:** If `INVALID_PDU` (status 4) occurs on the descriptor write,
the callback retries once. The connection state machine also retries once on
`GATT_INSUFFICIENT_AUTHORIZATION` during connection.

---

## 2. KISS Framing

All data on both the TX (write) and RX (notification) characteristics is framed
as KISS frames.

### Special Bytes

| Byte | Value | Role |
|------|-------|------|
| **FEND** | `0xC0` | Frame delimiter (start/end) |
| **FESC** | `0xDB` | Escape introducer |
| **TFEND** | `0xDC` | Escaped FEND inside data |
| **TFESC** | `0xDD` | Escaped FESC inside data |

### Encoding (Sender)

```
output.write(FEND)
for each byte b in payload:
    if b == FEND:   output.write(FESC); output.write(TFEND)
    else if b == FESC:  output.write(FESC); output.write(TFESC)
    else:           output.write(b)
output.write(FEND)
```

### Decoding (Receiver) — State Machine

```
VOID ──FEND──→ GET_CMD
GET_CMD ──FEND──→ (ignore, stay)
GET_CMD ──FESC──→ ESCAPE
GET_CMD ──data──→ GET_DATA
GET_DATA ──FEND──→ deliver frame; → GET_CMD
GET_DATA ──FESC──→ ESCAPE
GET_DATA ──data──→ (append, stay)
ESCAPE ──TFEND──→ append FEND; → GET_DATA
ESCAPE ──TFESC──→ append FESC; → GET_DATA
ESCAPE ──FEND──→ error (duplicate FEND), → VOID
ESCAPE ──FESC──→ error (duplicate FESC), → VOID
any state ──FEND──→ (duplicate FEND at VOID is ignored)
```

Frame payload (everything between the surrounding FEND bytes, after
unescaping) is passed to `TncViewModel.decodePacket()`.

**Buffer overflow:** The decoder has an initial 256-byte buffer that is
extended in 256-byte chunks if needed. If overflow occurs the frame is
discarded and state resets to VOID.

---

## 3. Standard KISS Parameters (Commands 0x01–0x05)

These are standard KISS commands. Payload is a single byte.

| Cmd | Name | Arg | Description |
|-----|------|-----|-------------|
| `0x01` | TXDELAY | `uint8` | Transmit delay in 10 ms units (0–255) |
| `0x02` | PERSISTENCE | `uint8` | Persistence parameter (0–255) |
| `0x03` | SLOTIME | `uint8` | Slot time in 10 ms units (0–255) |
| `0x04` | TXTAIL | `uint8` | TX tail in 10 ms units (0–255) |
| `0x05` | DUPLEX | `uint8` | `0` = half duplex, `1` = full duplex |

---

## 4. SetHardware Commands (Command 0x06)

Format: `[0x06, <subcmd>, <arg...>]`

A subcommand byte identifies the specific hardware parameter, followed by
variable-length argument data.

### 4.1 Single-Byte Argument Commands

| Subcmd | Constant Name | Arg | Description |
|--------|--------------|-----|-------------|
| `0x01` | `TNC_SET_OUTPUT_VOLUME` | `uint8` | Output volume |
| `0x02` | `TNC_SET_INPUT_ATTEN` | `uint8` | Input attenuation |
| `0x03` | `TNC_SET_SQUELCH_LEVEL` | `uint8` | Squelch level |
| `0x05` | `TNC_STREAM_VOLUME` | *(none)* | Stream volume |
| `0x06` | `TNC_GET_BATTERY_LEVEL` | *(none)* | Query battery level (response: cmd `0x06`, subcmd `0x06`) |
| `0x07` | `TNC_PTT_MARK` | *(none)* | PTT mark only |
| `0x08` | `TNC_PTT_SPACE` | *(none)* | PTT space only |
| `0x09` | `TNC_PTT_BOTH` | *(none)* | PTT mark + space |
| `0x0A` | `TNC_PTT_OFF` | *(none)* | PTT off / stop TX |
| `0x0C` | `TNC_GET_OUTPUT_VOLUME` | *(none)* | Query output volume |
| `0x10` | `TNC_SET_VERBOSITY` | `uint8` | Verbosity level |
| `0x18` | `TNC_SET_INPUT_TWIST` | `int8` | Input twist (signed byte) |
| `0x1A` | `TNC_SET_OUTPUT_TWIST` | `int8` | Output twist (signed byte) |
| `0x2A` | `TNC_SAVE_EEPROM` | *(none)* | Save current settings to EEPROM |
| `0x2B` | `TNC_ADJUST_INPUT_LEVELSval` | *(none)* | Trigger input level adjustment |
| `0x31` | `TNC_GET_DATETIME` | *(none)* | Query RTC datetime |
| `0x45` | `TNC_SET_BT_CONN_TRACK` | `uint8` | Bluetooth connection tracking |
| `0x49` | `TNC_SET_USB_POWER_ON` | `uint8` | USB power-on behavior |
| `0x4B` | `TNC_SET_USB_POWER_OFF` | `uint8` | USB power-off behavior |
| `0x4F` | `TNC_SET_PTT_CHANNEL` | `uint8` | PTT channel/style |
| `0x50` | `TNC_GET_PTT_CHANNEL` | *(none)* | Query PTT channel |
| `0x51` | `TNC_SET_PASSALL` | `uint8` | Pass-all mode (`0` or `1`) |
| `0x53` | `TNC_SET_RX_REVERSE_POLARITY` | `uint8` | RX polarity reverse (`0` or `1`) |
| `0x55` | `TNC_SET_TX_REVERSE_POLARITY` | `uint8` | TX polarity reverse (`0` or `1`) |
| `0x7F` | `TNC_GET_ALL_VALUES` | *(none)* | Query all configuration values |

### 4.2 16-Bit Signed Argument Commands

Format: `[0x06, <subcmd>, <high_byte>, <low_byte>]` — big-endian signed 16-bit.

| Subcmd | Constant Name | Description |
|--------|--------------|-------------|
| `0x01` | `TNC_SET_OUTPUT_GAIN` | Output gain, API 2.0 (16-bit signed) |

### 4.3 16-Bit Unsigned Argument Commands

Format: `[0x06, <subcmd>, <high_byte>, <low_byte>]` — big-endian unsigned 16-bit.

| Subcmd | Constant Name | Description |
|--------|--------------|-------------|
| `0x02` | `TNC_SET_INPUT_GAIN` | Input gain (16-bit unsigned) |

### 4.4 BCD Datetime Command

Format: `[0x06, 0x32, YY, MM, DD, DW, HH, MM, SS]`

BCD-encoded decimal (each nibble is one digit). DW is day-of-week (0=Sunday).

| Byte | Field | Format |
|------|-------|--------|
| 2 | YY | BCD, year − 2000 |
| 3 | MM | BCD, month (1–12) |
| 4 | DD | BCD, day of month |
| 5 | DW | `uint8`, day of week (0=Sun) |
| 6 | HH | BCD, hour |
| 7 | mm | BCD, minute |
| 8 | SS | BCD, second |

### 4.5 Extended Modem Type Command

Format: `[0x06, 0xC1, 0x82, <modem_type>]`

The byte `0xC1` (`0xFF` sign-extended as a signed byte = `−63`) is the escape that
indicates extended configuration. The following byte `0x82` is the extended
subcommand for setting the modem type.

| Bytes | Description |
|-------|-------------|
| `[0x06, 0xC1, 0x82, 0xNN]` | Set active modem type to `0xNN` |

---

## 5. Response Parsing

All TNC responses arrive as KISS frames on the RX characteristic (notification).
The decoder delivers the raw payload (after FEND/FESC processing, without the
delimiters) to `TncViewModel.decodePacket()`.

### 5.1 Top-Level Dispatch

```kotlin
when (packet[0]) {
    0x01 → tncTxDelay       = packet[1] as uint8
    0x02 → tncPersistence   = packet[1] as uint8
    0x03 → tncSlotTime      = packet[1] as uint8
    0x04 → tncTxTail        = packet[1] as uint8
    0x05 → tncDuplex        = packet[1] as int8
    0x06 → decodeHardwareConfiguration(packet)
}
```

### 5.2 Hardware Configuration (cmd 0x06) — subcmd byte index 1

General format: `[0x06, <subcmd>, <data...>]`

| subcmd | Field | Data Format | LiveData |
|--------|-------|-------------|----------|
| `0x04` | Input Level | `uint16be` | `tncInputLevel` |
| `0x06` | Battery Level | `uint16be` mV | `tncBatteryLevel` |
| `0x0C` | TX Gain | `uint16be` | `tncTxGain` |
| `0x0D` | RX Gain | `uint16be` | `tncInputGain` |
| `0x11` | Verbosity | `uint8` | `tncVerbosityLevel` |
| `0x19` | RX Twist | `int8` | `tncInputTwist` |
| `0x1B` | TX Twist | `int8` | `tncTxTwist` |
| `0x21` | TX Delay | `uint8` | `tncTxDelay` |
| `0x22` | Persistence | `uint8` | `tncPersistence` |
| `0x23` | Slot Time | `uint8` | `tncSlotTime` |
| `0x24` | TX Tail | `uint8` | `tncTxTail` |
| `0x25` | Duplex | `uint8` | `tncDuplex` |
| `0x28` | Firmware Version | UTF-8 string | `tncFirmwareVersion` |
| `0x29` | Hardware Version | UTF-8 string | `tncHardwareVersion` |
| `0x2F` | Serial Number | UTF-8 string | `tncSerialNumber` |
| `0x30` | MAC Address | 6-byte raw, formatted as `XX:XX:XX:XX:XX:XX` | `tncMacAddress` |
| `0x31` | DateTime | BCD YYMMDDWWHHMMSS (8 bytes) | `tncDateTime` |
| `0x46` | Connection Tracking | `uint8` | `tncConnectionTracking` |
| `0x4A` | USB Power On | `uint8` | `tncUsbPowerOn` |
| `0x4C` | USB Power Off | `uint8` | `tncUsbPowerOff` |
| `0x50` | PTT Style | `uint8` | `tncPttStyle` |
| `0x52` | Pass-All | `uint8` | `tncPassall` |
| `0x54` | RX Polarity | `uint8` | `tncRxPolarity` |
| `0x56` | TX Polarity | `uint8` | `tncTxPolarity` |
| `0x77` | Min TX Twist | `uint8` | `tncMinimumTxTwist` |
| `0x78` | Max TX Twist | `uint8` | `tncMaximumTxTwist` |
| `0x79` | Min RX Twist | `uint8` | `tncMinimumRxTwist` |
| `0x7A` | Max RX Twist | `uint8` | `tncMaximumRxTwist` |
| `0x7B` | API Version | `int8` | `tncApiVersion` |
| `0x7C` | Min RX Gain | `uint16be` | `tncMinimumRxGain` |
| `0x7D` | Max RX Gain | `uint16be` | `tncMaximumRxGain` |
| `0x7E` | Capabilities | `uint8` | `tncCapabilities` |
| `0xC1` (`0xFF` as signed) | Extended hardware config | — | — |

### 5.3 Extended Hardware Configuration — subcmd 0xC1 (signed −63)

Triggered when `packet[1] == 0xC1` (`−63` as signed byte). Format:
`[0x06, 0xC1, <ext_subcmd>, <data...>]`

| ext_subcmd | Field | Data Format | LiveData |
|-----------|-------|-------------|----------|
| `0x81` (`−127` signed) | Modem Type | `uint8` | `tncModemType` |
| `0x83` (`−125` signed) | Supported Modem Types | raw byte array | `tncSupportedModemTypes` |

### 5.4 BCD DateTime Decoding

```
Byte 0 → YY   from_bcd() = (byte>>4)*10 + (byte&0x0F), then +2000
Byte 1 → MM   from_bcd() − 1  (month is 0-indexed in Calendar)
Byte 2 → DD   from_bcd()
Byte 3 → DW   byte + 1  (Calendar uses 1=Sunday)
Byte 4 → HH   from_bcd()
Byte 5 → mm   from_bcd()
Byte 6 → SS   from_bcd()
```

Example output string: `2026-04-18T14:30:00 UTC`

---

## 6. Data Write Flow

Commands are sent by writing a KISS-encoded frame to the TX characteristic
(`WRITE_TYPE_NO_RESPONSE`). Because this is a write-without-response, the
`BLEOutputStream.flush()` uses a semaphore (`awaitingCallback`) to rate-limit:
it only sends the next chunk when the `onCharacteristicWrite` callback fires.

```kotlin
// BLEOutputStream.flush()
if (awaitingCallback.tryAcquire()) sendMore()

// send() chops buffer into mtu-sized chunks
private fun send(): Int {
    val chunk = buffer.take(mtu).toByteArray()
    buffer = buffer.drop(mtu).toByteArray()
    this@BluetoothLEService.send(chunk)   // → gatt.writeCharacteristic(tx, chunk)
    return chunk.size
}
```

**Practical implication:** To ensure a command reaches the TNC, caller must call
`outputStream.flush()` after writing. The stream will block sending if the
previous write's callback has not yet fired.

## Appendix: Quick Reference Tables

### BLE UUIDs
```
Service:        00000001-ba2a-46c9-ae49-01b0961f68bb
TX (write):     00000002-ba2a-46c9-ae49-01b0961f68bb
RX (notify):    00000003-ba2a-46c9-ae49-01b0961f68bb
CCCD:           00002902-0000-1000-8000-00805f9b34fb
```

### KISS Special Bytes
```
FEND  = 0xC0   (frame delimiter)
FESC  = 0xDB   (escape introducer)
TFEND = 0xDC   (escaped FEND)
TFESC = 0xDD   (escaped FESC)
```

### Command Byte Map
```
0x01  TXDELAY          (uint8)
0x02  PERSISTENCE      (uint8)
0x03  SLOTIME          (uint8)
0x04  TXTAIL           (uint8)
0x05  DUPLEX           (uint8)
0x06  Hardware config  (see subcmd table)
```

### Hardware Subcommand Quick Ref (cmd 0x06)
```
0x01  SET_OUTPUT_VOLUME    uint8     | 0x01  SET_OUTPUT_GAIN     int16be
0x02  SET_INPUT_ATTEN      uint8     | 0x02  SET_INPUT_GAIN      uint16be
0x03  SET_SQUELCH_LEVEL    uint8     | 0x18  SET_INPUT_TWIST     int8
0x05  STREAM_VOLUME        —         | 0x1A  SET_OUTPUT_TWIST    int8
0x06  GET_BATTERY_LEVEL    —         | 0x2A  SAVE_EEPROM          —
0x07  PTT_MARK             —         | 0x2B  ADJUST_INPUT_LEVELS —
0x08  PTT_SPACE            —         | 0x31  GET_DATETIME         —
0x09  PTT_BOTH             —         | 0x32  SET_DATETIME        BCD 9 bytes
0x0A  PTT_OFF              —         | 0x45  SET_BT_CONN_TRACK   uint8
0x0C  GET_OUTPUT_VOLUME    —         | 0x49  SET_USB_POWER_ON    uint8
0x10  SET_VERBOSITY        uint8     | 0x4B  SET_USB_POWER_OFF   uint8
0x7F  GET_ALL_VALUES       —         | 0x4F  SET_PTT_CHANNEL     uint8
                                 | 0x50  GET_PTT_CHANNEL     —
0x51  SET_PASSALL          uint8     | 0x53  SET_RX_REVERSE_POL  uint8
0x55  SET_TX_REVERSE_POL  uint8     | 0xC1  Extended (see §5.3)
```
