# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## 6.0.2 – 2026-07-21

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
