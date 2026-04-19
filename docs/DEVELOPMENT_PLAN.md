# Mobilinkd Web Configurator — Development Plan

## Phase 1: BLE Discovery, Connection, Disconnection & Error Handling
**Goal:** Reliable BLE lifecycle — find TNC, pair, connect, disconnect, handle errors gracefully.

- [ ] `scan()` — call `navigator.bluetooth.requestDevice()` with correct filters
- [ ] `connect(dev)` — GATT connect, get service + characteristics, enable notifications
- [ ] `disconnect()` — graceful disconnect, cleanup
- [ ] `gattserverdisconnected` event — auto-reconnect or show disconnected state
- [ ] Error handling — `NotFoundError` (user cancelled), `SecurityError` (not paired), `NetworkError`, timeout
- [ ] UI state machine — `disconnected → scanning → connecting → connected → error`
- [ ] Connection card updates in real-time (name, MAC if available)
- [ ] Disable/enable buttons appropriately per state

**Status:** Not started

---

## Phase 2: SLIP (KISS Framing)
**Goal:** Reliable byte framing for BLE GATT.

- [x] `KissEncoder.encode(bytes)` — SLIP encode with FEND/FESC/TFEND/TFESC (already done)
- [x] `KissDecoder` state machine — VOID → GET_CMD → GET_DATA → ESCAPE states
- [x] Buffer overflow handling (≥512 bytes → reset + discard)
- [x] Deliver complete frames to `onKissData()` as `Uint8Array`
- [x] `onKissData()` → frame router (`decodePacket`)

**Status:** Complete

---

## Phase 3: Hardware Command Encoder & Decoder
**Goal:** Bidirectional command/response protocol.

**Encoder:**
- [x] `tx()` — already sends raw KISS frames (cmd 0x06 subcmd encoding done inline)
- [x] `sendGetAllValues()` → `tx(0x06, 0x7F)`
- [ ] `sendSaveEeprom()` → `tx(0x06, 0x2A)`
- [ ] `sendSetTxDelay(v)`, `sendSetPersistence(v)`, `sendSetSlotTime(v)`, `sendSetDuplex(v)`
- [ ] `sendSetOutputGain(v)` (16-bit), `sendSetInputGain(v)` (16-bit)
- [ ] `sendSetInputTwist(v)`, `sendSetOutputTwist(v)` (int8)
- [ ] `sendSetPttChannel(v)`, `sendStreamStart()`, `sendStreamStop()`

**Decoder:**
- [x] `decodePacket(frame)` — route by cmd byte (0x01–0x05 KISS, 0x06 hardware)
- [x] `decodeHardwareConfig(data)` — route by subcmd byte (all subcmds 0x04–0x7E)
- [x] String fields (firmware, hardware, serial, MAC) parsing
- [x] BCD datetime decoding
- [x] Extended subcmd 0xC1 routing
- [x] Store decoded values in `tncState` object; update UI immediately on each response

**Status:** Decoder complete; encoder partial (GET_ALL_VALUES done, individual setters not yet wired to UI)

---

## Phase 4: TNC Info Screen
**Goal:** Display TNC identity on connect.

- [ ] On `connect()` → `sendGetAllValues()` → parse responses
- [ ] Populate: Hardware Version, Firmware Version, Serial Number, MAC Address, API Version
- [ ] Display DateTime, update battery bar
- [ ] `refreshInfo()` button (re-query)

**Status:** Not started

---

## Phase 5: KISS Parameters Screen (TXDELAY, PERSIST, SLOTTIME, DUPLEX)
**Goal:** Standard KISS config.

- [ ] `sendGetKissParams()` — query current values
- [ ] Parse and populate range sliders + displays
- [ ] `sendSetTxDelay(v)`, `sendSetPersistence(v)`, `sendSetSlotTime(v)`, `sendSetDuplex(v)`
- [ ] Save/Revert buttons → `sendSaveEeprom()` / re-query

**Status:** Not started

---

## Phase 6: Modem Configuration Screen
**Goal:** Modem type selection.

- [ ] `sendGetModemType()` → extended 0xC1/0x81 query
- [ ] `sendSetModemType(type)` → extended 0xC1/0x82 write
- [ ] Dropdown populated from `GET_ALL_VALUES` supported-modems list
- [ ] Save → `sendSaveEeprom()`

**Status:** Not started

---

## Phase 7: Receive Audio Screen
**Goal:** RX streaming and audio level display.

- [ ] `sendStreamStart()` / `sendStreamStop()` → TX 0x06/0x05 and 0x06/0x0A
- [ ] Display RX audio level (if TNC sends stream data)
- [ ] Update RX mode badge (idle/streaming)
- [ ] Audio exclusive notice (TX + RX mutual exclusivity)

**Status:** Not started

---

## Phase 8: Transmit Audio Screen
**Goal:** TX gain and twist, PTT controls.

- [ ] `sendSetOutputGain(v)` (16-bit) and `sendSetOutputTwist(v)` (int8)
- [ ] Populate TX gain/twist range displays
- [ ] PTT buttons (Off/Mark/Space/Both) → `sendPttMark/Space/Both/Off()`
- [ ] PTT style chips (Simplex/Multiplex) → `sendSetPttChannel(v)`
- [ ] Audio exclusive notice
- [ ] Save → `sendSaveEeprom()`

**Status:** Not started

---

## Phase 9: Power Settings Screen
**Goal:** USB power and BT connection behavior.

- [ ] Query: USB Power On, USB Power Off, Connection Tracking
- [ ] Set/Get for each
- [ ] Save → `sendSaveEeprom()`

**Status:** Not started

---

## Phase 10: Protocol Log View (diagnostic)
**Goal:** See all raw KISS frames for debugging.

- [ ] Log all TX frames
- [ ] Log all RX frames
- [ ] Color coding: green = handled, yellow = valid/unhandled, red = invalid/overflow
- [ ] Visible by default during development

**Status:** Not started

---

## Not in Scope (hardware not supported)
- Auto-adjust input levels
- TX Tail
- Squelch Level
- Connection Tracking
