# Changelog

All notable changes to this patch are documented here.

## [1.0.0] — 2025

### Added
- `patch/BLERemoteDescriptor.cpp` — patched ArduinoBLE file with `writeCmd` substitution
- `patch/arduinoble_myo_cccd.patch` — standard unified diff for manual application
- `docs/TECHNICAL_NOTES.md` — full ATT opcode analysis and failure mode documentation

### Fixed
- `BLERemoteDescriptor::writeValue()` now uses `ATT.writeCmd()` (opcode `0x52`, write-without-response) instead of `ATT.writeReq()` (opcode `0x12`, blocking write request)
- Resolves silent subscription failure on Myo Armband and any BLE peripheral that does not implement ATT Write Response for descriptor handles
- Resolves secondary deadlock caused by `writeReq` competing for ATT channel slots with high-throughput notification streams (250+ packets/sec)

### Affected upstream file
`ArduinoBLE/src/remote/BLERemoteDescriptor.cpp` — method `writeValue()`, line ~52  
Upstream library version: ArduinoBLE v1.3.x  
Upstream issue: [ArduinoBLE #71](https://github.com/arduino-libraries/ArduinoBLE/issues/71)
