# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## 6.0.3 – 2026-11-16

### Added – DrvSigPccc
- **Zero confirmation filter** – New line option `ZeroConfirmCycles` (Extra Config, default 0 = off) keeps the previous value of a numeric tag that drops from non-zero to exactly 0 until 0 has been read that many further cycles in a row. A read group in the device template can override it (`zeroConfirmCycles`, "Override zero confirm cycles"), so the filter can be limited to the tags of an affected rack or module. Every held-back zero and its outcome (discarded or confirmed) is logged. Intended for PLC-5 data that briefly reads 0 while ladder logic or a block transfer updates it.

### Changed – DrvSigPccc
- **Reconnect only on link errors** – A single timeout or an error reported by the PLC (wrong address, file size) now fails only that read group instead of closing the connection and invalidating every tag. Repeated timeouts still open the circuit breaker and trigger a reconnect.
- **PCCCComm 1.1.0** – DF1 frame timeout scaled to the baud rate, short or empty read replies rejected instead of being read as 0, and status codes carried in exceptions.

### Fixed – DrvSigPccc
- **Timer/Counter/Control sub-elements on PLC-5** – Merged reads of `.PRE`/`.ACC`/`.LEN`/`.POS` used a ×3 offset and returned the wrong element (e.g. `T4:153.PRE` returned `T4:155.PRE`).
- **Circuit breaker never tripped** – The connection was reopened before every request, which reset the breaker, so a half-open TCP connection was never recovered.
- **Unsigned integers** – Integer words are interpreted per data type: a `UShort` above 32767 no longer reads as negative.
- **32-bit writes** – Values are rounded and range-checked and sent as the exact bit pattern instead of a truncating cast.
- **Bit groups** – A reply with fewer words than requested now fails the group instead of reading the missing bits as 0.
- **SLC/MicroLogix bit writes** – Use Masked Write (FNC 0xAB), with fallback to the previous method when the controller does not support it.
- **Template editor** – A failed template load no longer leaves the device bound to an empty template; translated labels fit their fields.

### Fixed – DrvSig7
- **Empty element group** – An empty group no longer aborts device initialization with an overflow error.
- **Bool commands** – Bit writes send `0x00`/`0x01`.
- **Integer commands** – Values are rounded instead of truncated; NaN is rejected.
- **Reconnect on ISO framing errors** – Invalid PDU, invalid data size and invalid PLC answers now reconnect instead of reading misaligned data.
- **Permanent errors** – Reads and commands that fail deterministically (bad address, item not available, size mismatch) are no longer retried.
- **Quieter logging** – Read timing is logged on the first cycle and after recovery only, or always with the line option `VerboseLog`.
- **Template editor** – Tag numbers are refreshed after deleting, moving or deactivating a group; empty groups and overlapping elements are reported; a failed load can no longer overwrite the original file; translated labels fit their fields.

### Fixed – DrvSigModbus
- **WriteSingleRegister ignored scaling** – A `UShort`/`Short` element with scaling wrote the engineering value as the raw register, and negative `Short` values were sent as 0 on .NET 9 and later.
- **64-bit values with byte order** – `ULong`, `Long` and `Double` elements with a byte order were decoded incorrectly.
- **Short command data** – Multiple-register and coil writes whose data is shorter than the request are padded with zeros again instead of failing.
- **Invalid values** – NaN, infinity and `Float` values beyond its range are rejected; a malformed scaling string no longer stops polling, commands or template saving.
- **Template editor** – An invalid scaling text is kept as typed and marked instead of silently keeping the previous value; scaling is disabled for bit-mask elements; the scaling example fits its field.

---

## 6.0.2 – 2026-07-21

### Added – DrvSigModbus
- **Scaling for all numeric types** – Scaling is now supported for `UInt`, `Int`, `ULong`, `Long`, `Float`, and `Double` (previously limited to `UShort` and `Short`). Scaling is explicitly excluded for `Bool` and `Undefined` types.

### Changed – DrvSigModbus
- **Optimize for .NET 8** – Common and Logic projects now target `net8.0` (previously `netstandard2.0`), unlocking performance improvements and modern APIs.
- **Span‑based operations** – Replaced array-based operations with `Span<T>` and `ReadOnlySpan<byte>` for CRC16/LRC calculations, byte-order reordering (`ApplyByteOrder`), and data conversion, reducing heap allocations.
- **Parsed scaling cache** – Scaling strings are now parsed once and cached per element/command instance, eliminating repeated parsing during polling.
- **Zero‑allocation byte ordering** – `ElemGroup.GetElemVal` now uses direct byte mapping according to `byteOrder`, avoiding intermediate array allocations.
- **BinaryPrimitives for commands** – `ModbusCmd.SetCmdData` writes data directly in big‑endian using `BinaryPrimitives.WriteXxxBigEndian`, removing the need for `Array.Reverse`.
- **Convert.ToHexString in ASCII mode** – Replaced manual `StringBuilder` loop with `Convert.ToHexString` for building request strings in ASCII mode.
- **CRC tables as ReadOnlySpan** – CRC lookup tables are now static `ReadOnlySpan<byte>` properties, preventing heap allocation at static construction.
- **ArgumentNullException.ThrowIfNull** – Replaced explicit null checks with the modern, concise `ThrowIfNull` pattern.
- **Remove redundant Array.Clear** – Removed unnecessary `Array.Clear` call in `ElemGroup.InitReqPDU` (new `byte[]` is already zero‑initialized).
- **Initialize Data in InitReqPDU** – Ensured `Data` is initialized with a zeroed array if null, preventing `ArgumentNullException` when commands are created from element configurations.
- **Improve ULong/Long range validation** – Replaced `double` comparisons with `decimal` (with overflow handling) for accurate boundary checks in `ModbusCmd.SetCmdData`.

### Changed – DrvSigPccc
- **CheckSumOptions** – Updated namespace from `PCCC.Pccc` to `PCCC.Core` to align with PCCCComm library restructuring.

### Fixed – DrvSigPccc
- **Connection recovery deadlock** – Resolved issue where `_connBroken` flag prevented subsequent reconnect attempts after communication failure. Added `TryReconnect()` method called at session start to break the deadlock and allow cooldown expiry checks.

---

## 6.0.1 – 2026-07-11

### Added – All Drivers
- **.NET 8 migration** – All projects migrated from `netstandard2.0` to `net8.0` for improved performance and modern language features.
- **Solution reorganization** – Projects grouped into logical solution folders for better navigation.

### Added – DrvSigPccc
- **Circuit Breaker & Bulkhead** – Integrated PCCCComm's circuit breaker and request throttling to prevent thread pool exhaustion during communication failures.
- **Masked ReadModifyWrite for bit writes** – Bit-addressed writes (`B3:0/5`, `T4:0/EN`) now use atomic ReadModifyWrite operations instead of read‑then‑write, eliminating race conditions.
- **Skip remaining groups on connection failure** – When a read group fails, subsequent groups in the same poll cycle are skipped to avoid unnecessary timeouts and speed up recovery.

### Fixed – DrvSigPccc
- **Byte order and scaling order** – Corrected execution order so byte order is applied before scaling (matching Modbus convention).
- **Single‑bit byte‑order skip** – Byte‑order reordering is no longer applied to single‑bit elements (e.g., `B3:0/5`).
- **Template validation** – Improved element generation and validation to reject invalid addresses and data types at load time.
- **ReadGroup batching semantics** – Updated comments to reflect that template groups are UI containers only; physical batching is global and based on address proximity.

### Changed – DrvSigPccc
- **UI standardization** – Standardized control sizes and layouts in device template editor across all drivers.

### Changed – DrvSig7
- **UI updates** – Resolved designer compatibility issues and updated forms for modern .NET 8.

### Changed – DrvSigModbus
- **Solution folder** – Added to solution folder for better organization.

---

## 6.0.0 – 2026-07-02

### Added – Initial Release

- **DrvSigModbus** – Enhanced Modbus driver with driver‑level scaling (Free, unlimited).
- **DrvSigPccc** – Allen‑Bradley PCCC driver (SLC, MicroLogix, PLC‑5).
- **DrvSig7** – Siemens S7 driver (S7‑300, S7‑400, S7‑1200, S7‑1500).
- Commercial licensing system with hardware‑locked tiers:
  - **Trial**: 10 tags (no time limit).
  - **Personal**: 100 tags.
  - **Professional**: 500 tags.
  - **Enterprise**: Unlimited tags.
- Example templates for all three drivers.
- Documentation: README, licensing guide, and template guides.

### Known Issues
- None reported for this release.

---
