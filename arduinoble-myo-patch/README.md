# ArduinoBLE Myo Patch — `ATT.writeReq()` → `ATT.writeCmd()`

> **One-line fix that makes Myo EMG streaming work on bare-metal Arduino.**  
> Without this patch, you will receive **zero EMG notifications** — no errors, no timeouts, just silence.

---

## The Problem

You have successfully connected your Arduino Nano 33 BLE (or any nRF52840 board) to a Myo armband. The device is paired. Subscription calls return without error. And yet — **no EMG data ever arrives**.

The root cause is a single wrong ATT opcode deep inside ArduinoBLE.

### How Myo EMG activation works

To receive EMG notifications from the Myo, the BLE Central must write `0x0100` to each EMG characteristic's **CCCD descriptor** (UUID `0x2902`). This is standard BLE subscription — nothing unusual so far.

The problem is *how* that write is issued. The Myo's GATT stack requires a BLE ATT **Write Command** — opcode `0x52`, *write-without-response*. It does **not** implement ATT Write Response (`0x13`) for this descriptor.

ArduinoBLE v1.3.x routes **all** CCCD writes through `ATT.writeReq()` — a **Write Request** (opcode `0x12`) that blocks indefinitely waiting for a Write Response that the Myo will never send.

```
BLE Central                         Myo Peripheral
     │                                    │
     │── ATT_WRITE_REQ (0x12) ──────────►│  ← ArduinoBLE sends this
     │                                    │    Myo ignores / does not respond
     │◄ [waiting for ATT_WRITE_RSP] ──────│  ← This response never comes
     │        ⏳ blocks forever           │
     │                                    │  ← No subscription confirmed
     │                                    │  ← No notifications ever sent
```

The function hangs. The subscription is never confirmed. Zero notifications arrive. Ever.

---

## The Fix

One line in `ArduinoBLE/src/remote/BLERemoteDescriptor.cpp`, inside `writeValue()`:

```cpp
// BEFORE — blocks permanently on Myo (waits for ATT_WRITE_RSP that never comes):
ATT.writeReq(_connectionHandle, _handle, value, length);

// AFTER — write-without-response, fires and returns immediately:
ATT.writeCmd(_connectionHandle, _handle, value, length);
```

`ATT.writeCmd()` issues opcode `0x52` — a Write Command. It requires no response, returns immediately, and is exactly what the Myo expects.

---

## Installation

### Option A — Copy the patched file (recommended)

```bash
# Windows
cp patch/BLERemoteDescriptor.cpp "%USERPROFILE%\Documents\Arduino\libraries\ArduinoBLE\src\remote\BLERemoteDescriptor.cpp"

# Linux / macOS
cp patch/BLERemoteDescriptor.cpp ~/Arduino/libraries/ArduinoBLE/src/remote/BLERemoteDescriptor.cpp
```

Then recompile your sketch. No other changes needed.

### Option B — Apply the diff manually

Edit `ArduinoBLE/src/remote/BLERemoteDescriptor.cpp` and find `writeValue()`. Replace the `ATT.writeReq(...)` call with `ATT.writeCmd(...)`. The rest of the file is unchanged.

---

## Why `writeCmd` is safe here

`ATT.writeReq()` is appropriate when the peripheral confirms receipt (e.g., writing a configuration register on a sensor that acknowledges writes). For CCCD subscription on the Myo, the armband silently activates the stream — it does not send a Write Response. Using `writeCmd` matches the Myo's actual ATT implementation.

**Impact on other devices:** `writeCmd` is valid per the BLE specification for any peripheral that supports it. For peripherals that *require* a Write Response, `writeCmd` may not guarantee delivery on noisy channels — but CCCD descriptors on consumer BLE devices almost universally support write-without-response.

If you need strict compatibility with both device classes, the safer production fix is to detect the peripheral type and route accordingly. For Myo-specific deployments, this single substitution is sufficient.

---

## Background: Why this was never fixed upstream

Every prior Myo-based research system — without exception — uses an **OS-level BLE host stack**:

| Platform | BLE Stack | Handles opcode routing? |
|---|---|---|
| Linux | BlueZ | ✓ Kernel-managed |
| Windows | WinBLE | ✓ Kernel-managed |
| macOS / iOS | CoreBluetooth | ✓ Kernel-managed |
| Android | Android BLE API | ✓ Kernel-managed |
| Raspberry Pi | BlueZ | ✓ Kernel-managed |

These stacks absorb the Myo's ATT quirks transparently. Application code calls `subscribe()` and data flows — the correct opcode is handled internally.

ArduinoBLE is a **userspace BLE implementation** running on bare metal. It has no kernel buffer, no interrupt-driven notification queue, and no abstraction over the write opcode. When `writeReq` hangs, the entire firmware hangs with it.

**To our knowledge, this is the first documented solution enabling native Myo EMG streaming on a bare-metal MCU.** The issue has been reported (ArduinoBLE #71) but was not resolved in the library.

---

## Scope of this patch

This repository contains **only** the library fix:

```
arduinoble-myo-patch/
├── README.md                    ← This file
├── patch/
│   └── BLERemoteDescriptor.cpp  ← Patched ArduinoBLE file
└── docs/
    └── TECHNICAL_NOTES.md       ← ATT opcode reference and deeper analysis
```

This patch is extracted from a larger research project on adaptive TinyML-based sEMG gesture recognition for wheelchair control. That project's full firmware, data pipeline, and model are maintained separately.

---

## Tested with

| Component | Version / Model |
|---|---|
| Board | Arduino Nano 33 BLE Sense Lite (Nordic nRF52840) |
| ArduinoBLE | v1.3.x |
| Arduino IDE | 2.x |
| Myo Armband | Thalmic Labs (all hardware revisions) |

---

## Related issue

[ArduinoBLE GitHub Issue #71](https://github.com/arduino-libraries/ArduinoBLE/issues/71) — `writeValue()` on remote descriptors blocks indefinitely on devices that do not send Write Response.

---

## Citation

If this patch saves you days of debugging, please cite:

```bibtex
@misc{rabby2025arduinoble,
  title        = {ArduinoBLE Myo Patch: ATT Write Command Fix for Bare-Metal EMG Streaming},
  author       = {Rabby, Fazlay},
  year         = {2025},
  howpublished = {\url{https://github.com/fazlayofficial/arduinoble-myo-patch}},
  note         = {Enables native Myo BLE EMG streaming on Arduino Nano 33 / nRF52840 
                  by replacing ATT.writeReq() with ATT.writeCmd() in BLERemoteDescriptor.cpp}
}
```

---

## Author

**Fazlay Rabby** — Embedded AI & Assistive Robotics  
📧 fazlay.rabby@ieee.org  
🔗 [LinkedIn](https://www.linkedin.com/in/fazlayrabbyofficial/)  
🎓 [Google Scholar](https://scholar.google.com/citations?hl=en&user=DhNowuUAAAAJ)  
🌐 [Portfolio](https://fazlayofficial.github.io/)
