# Technical Notes — ATT Opcode Analysis and Patch Rationale

## BLE ATT Write Opcodes: The Difference That Matters

The Bluetooth ATT (Attribute Protocol) layer defines two distinct write mechanisms:

| Operation | Opcode | Response expected? | ArduinoBLE function |
|---|---|---|---|
| Write Request | `0x12` | Yes — `ATT_WRITE_RSP (0x13)` | `ATT.writeReq()` |
| Write Command | `0x52` | No | `ATT.writeCmd()` |

`writeReq` is a **synchronous, reliable** write. The sender blocks until the peripheral confirms receipt.  
`writeCmd` is a **fire-and-forget** write. The sender returns immediately; no confirmation is expected.

Both are valid under the BLE Core Specification. Which one a peripheral supports for a given attribute is defined by the attribute's **properties bitmask** in the GATT service definition — specifically the `WRITE_WITHOUT_RESPONSE` property bit.

---

## Why the Myo Requires `writeCmd` for CCCD

The Myo's GATT firmware marks its CCCD descriptors as supporting Write Command (`0x52`) only. It does not implement the ATT Write Response handler for these handles.

This is an unusual but not technically incorrect implementation. The BLE specification permits peripherals to omit Write Response support on descriptors. Most consumer devices do respond to descriptor writes because it is the safer default — but the Myo does not.

**ArduinoBLE's assumption:** `BLERemoteDescriptor::writeValue()` unconditionally calls `ATT.writeReq()` for all remote descriptor writes, including CCCDs. This is wrong for any peripheral that does not respond to Write Requests on its descriptors.

---

## What Happens Without the Patch

```
1. Arduino calls characteristic.subscribe()
2. ArduinoBLE internally calls BLERemoteDescriptor::writeValue(0x0100, 2) on the CCCD
3. writeValue() calls ATT.writeReq(_connectionHandle, _handle, value, 2)
4. ATT layer sends: [ATT_WRITE_REQ | handle_lo | handle_hi | 0x01 | 0x00]
5. Myo receives the Write Request
6. Myo does not send ATT_WRITE_RSP (not implemented for this handle)
7. ATT.writeReq() spins waiting for the response
8. No response ever arrives
9. subscribe() never returns / hangs indefinitely
10. No CCCD is written → no subscription confirmed → zero EMG notifications
```

On a multi-threaded OS with an async BLE stack, this hang would surface as a timeout error. On bare-metal ArduinoBLE, the firmware simply never proceeds past step 9.

---

## What Happens With the Patch

```
1. Arduino calls characteristic.subscribe()
2. ArduinoBLE internally calls BLERemoteDescriptor::writeValue(0x0100, 2) on the CCCD
3. writeValue() calls ATT.writeCmd(_connectionHandle, _handle, value, 2)  ← patched
4. ATT layer sends: [ATT_WRITE_CMD | handle_lo | handle_hi | 0x01 | 0x00]
5. ATT.writeCmd() returns immediately — no waiting
6. subscribe() returns true
7. Myo receives the Write Command, activates EMG notifications on that characteristic
8. EMG BLE notifications begin flowing at ~200 Hz from the Myo
9. Arduino receives and processes notifications via BLE.poll()
```

---

## The High-Throughput Interaction

A secondary reason `writeCmd` is necessary goes beyond just the missing Write Response. The Myo streams EMG data at up to **250 notifications/second** across 4 EMG characteristics simultaneously. This creates a saturated ATT channel.

On a saturated channel, even if the Myo *were* to send Write Responses, there is no guarantee those response packets would arrive before `writeReq`'s internal timeout (if one exists) or before the waiting loop is pre-empted. `writeCmd` eliminates this race entirely.

This is noted in the patch comment header in `BLERemoteDescriptor.cpp`:

> *"writeReq blocks waiting for ATT_WRITE_RSP. When the Myo is streaming 250+ notifications/sec those fill the ATT slot and the response never arrives, causing subscribe() and every keepalive to silently fail."*

---

## Affected ArduinoBLE Versions

Confirmed affected: **ArduinoBLE v1.3.x** (all patch revisions as of 2025).  
The issue originates in the initial implementation of `BLERemoteDescriptor` and has not been addressed upstream.

Reference issue: [ArduinoBLE #71](https://github.com/arduino-libraries/ArduinoBLE/issues/71)

---

## Generalization: Other Devices Affected by the Same Bug

Any BLE peripheral that does not send ATT Write Responses for descriptor writes will silently fail to subscribe under stock ArduinoBLE. The Myo is the highest-profile known case, but the underlying bug is general.

If you are experiencing silent subscription failures with non-Myo BLE devices, this patch may also resolve your issue — especially if the peripheral is a high-throughput streamer (wearable sensors, custom BLE peripherals, industrial sensor nodes).

---

## What This Patch Does NOT Change

- `BLERemoteDescriptor::read()` — unchanged, still uses `ATT.readReq()` (correct)
- `BLERemoteCharacteristic::writeValue()` — separate file, not touched
- Any peripheral (server-side) ArduinoBLE code — not affected
- BLE connection management, scanning, service discovery — not affected

The patch is **minimally invasive**: one function call substitution in one method in one file.
