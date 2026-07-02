# Rapid SCADA Communication Drivers

![Rapid SCADA](https://img.shields.io/badge/Rapid%20SCADA-6.0-blue)
![License](https://img.shields.io/badge/License-Commercial%20%2B%20Free-yellow)

A collection of communication drivers for [Rapid SCADA](https://rapidscada.org/) v6.0.

---

## Available Drivers

| Driver | Protocol / Devices | Key Feature | Status |
|--------|-------------------|-------------|--------|
| **DrvSigModbus** | Modbus RTU / TCP | Built-in Modbus enhanced with **driver‑level scaling** | ✅ **Free** (Unlimited) |
| **DrvSigPccc** | Allen‑Bradley PCCC (SLC, MicroLogix, PLC-5) via DF1, EtherNet/IP, or CSPv4 | Full read/write with **read‑group aggregation** | 🔒 **Trial** (max 10 tags) |
| **DrvSig7** | Siemens S7 (S7‑300, S7‑400, S7‑1200, S7‑1500) via ISO‑on‑TCP (Sharp7) | Full read/write with **multi‑area polling** | 🔒 **Trial** (max 10 tags) |

---

## ⚠️ CAUTION – DEVELOPMENT DRIVER

**These drivers are still under active development.**

They can perform **write operations** to PLC memory (e.g., commands, setpoints, mode switching) as defined in your device templates.

- Always **test your configuration on a test bench** before connecting to a production PLC.
- Review your template carefully – especially write commands and scaling parameters.
- The example templates provided are for **illustration only** and may not be suitable for your specific equipment.

**You are fully responsible for any consequences resulting from the use of these drivers on live systems.**

---

## 📦 Installation

1. Download the compiled binaries (`.dll` files) from the **Releases** tab.
2. Copy all `.dll` files to your Rapid SCADA `ScadaComm` and `ScadaAdmin` directory.
3. Register the desired drivers in your project's **Configuration Database** → **Secondary Tables** → **Device Types**:

   ```xml
   <DevTypes>
     <DevType>
       <DevTypeID>1002</DevTypeID>
       <Name>Sig Modbus</Name>
       <Driver>DrvSigModbus</Driver>
     </DevType>
     <DevType>
       <DevTypeID>1003</DevTypeID>
       <Name>Sig PCCC</Name>
       <Driver>DrvSigPccc</Driver>
     </DevType>
     <DevType>
       <DevTypeID>1004</DevTypeID>
       <Name>Sig S7</Name>
       <Driver>DrvSig7</Driver>
     </DevType>
   </DevTypes>
   ```

   **Note**: Adjust `DevTypeID` values to avoid conflicts with existing device types in your project.

4. **Configure** your communication lines and devices using the registered device types.
5. **Transfer** the configuration to the server.
6. **Restart** the Rapid SCADA communication service.

---

## 📝 Driver-Specific Documentation & Template Guides

### 🔹 DrvSigModbus – Driver‑Level Scaling

Unlike the standard Rapid SCADA Modbus driver, **DrvSigModbus** allows you to define scaling parameters directly inside the device template XML.

**The design is inspired by Citect SCADA**, where scaling and engineering units are configured directly at the tag level – eliminating the need for separate scaling calculations in the SCADA server and simplifying template maintenance.

**Scaling Format**: `"minRaw;maxRaw;minEng;maxEng"`

| Parameter | Description |
|-----------|-------------|
| `minRaw` | Minimum raw value from the PLC (e.g., `0`) |
| `maxRaw` | Maximum raw value from the PLC (e.g., `65535` for 16‑bit word) |
| `minEng` | Minimum engineering value displayed in SCADA |
| `maxEng` | Maximum engineering value displayed in SCADA |

**Example**:
```xml
<!-- 0-65535 raw → 0-100 engineering units -->
<Elem type="ushort" scaling="0;65535;0;100" tagCode="t0" name="Temperature" />

<!-- 0-4095 raw → 0-100 engineering units (12‑bit ADC) -->
<Elem type="ushort" scaling="0;4095;0;100" tagCode="pressure" name="Pressure" />
```

**Formula**: `Engineering = (Raw - minRaw) × (maxEng - minEng) / (maxRaw - minRaw) + minEng`

**Supported Data Blocks**:
- `Coils` (read/write)
- `HoldingRegisters` (read/write)
- `InputRegisters` (read‑only)
- `DiscreteInputs` (read‑only)

**Sample Template**: See `DrvSigModbus_Example.xml` in the repository.

---

### 🔹 DrvSigPccc – Allen‑Bradley PCCC Driver

Optimized for PCCC controllers with **automatic read‑group aggregation** for maximum performance.

**Key Features**:
- ✅ **Physical Batching**: All `<Elem>` elements across all active `<ReadGroup>` containers are globally flattened and sorted by physical address.
- ✅ **MaxMergeGap**: Controlled via line option `MaxMergeGap` (default: `8`). Larger values = more aggressive merging (fewer requests, but more wasted slots).
- ✅ **Automatic Bit Detection**: Bit-addressed elements (e.g., `"B3:0/5"`) are automatically detected and handled efficiently.
- ✅ **Structured Addressing**: Supports Timer (`T4:0.PRE`, `T4:0.ACC`) and Counter (`C5:0.PRE`, `C5:0.ACC`) sub-elements.

**Data Type Mapping**:

| Data Type | Description |
|-----------|-------------|
| `ushort` | 16‑bit unsigned integer (word) |
| `short` | 16‑bit signed integer |
| `int` / `long` | 32‑bit signed integer (L file) |
| `float` | 32‑bit IEEE 754 floating‑point (F file) |
| `string` | SLC/MicroLogix string file (ST file, 84 bytes) |
| `bool` | Single bit (extracted from word/bit addressing) |

**Example Configuration**:
```xml
<ReadGroup active="true" name="Integer N7 (UShort)">
  <Elem name="N7_0" tagCode="N7_0" address="N7:0" dataType="ushort" scaling="0;65535;0;100" />
  <Elem name="N7_1" tagCode="N7_1" address="N7:1" dataType="ushort" />
</ReadGroup>

<ReadGroup active="true" name="Timer T4 PRE">
  <Elem name="T4_0_PRE" tagCode="T4_0_PRE" address="T4:0.PRE" dataType="ushort" />
  <Elem name="T4_1_PRE" tagCode="T4_1_PRE" address="T4:1.PRE" dataType="ushort" />
</ReadGroup>
```

**Sample Template**: See `DrvSigPccc_Example.xml` in the repository.

---

### 🔹 DrvSig7 – Siemens S7 Driver

Supports both **ReadArea** and **ReadMultiVars** polling modes for efficient data collection.

**Key Features**:
- ✅ Supports all S7 data types: `Bool`, `Byte`, `Int`, `Word`, `DInt`, `DWord`, `Real`, `LReal`, `String`
- ✅ **Multi‑Area Polling**: Elements from the same DB (or I/O area) are placed in a single group to minimize round‑trips (one `ReadArea` call per DB).
- ✅ **Efficient Batching**: All active elements within the same DB are merged into a single contiguous read block.
- ✅ **Bit Addressing**: Supports `bitIndex` for packed BOOL values inside a single byte/word.
- ✅ **String Support**: Reads/writes S7 `STRING` type with configurable maximum length (`strLen`).

**Data Areas**:

| Area | Description |
|------|-------------|
| `DB` | Data Block (requires `dbNumber`) |
| `PE` | Process Image Inputs (S7‑300/400: PIW/PIB) |
| `PA` | Process Image Outputs (S7‑300/400: PQW/PQB) |

**Example Configuration**:
```xml
<!-- Data Block with mixed types -->
<ElemGroup active="true" name="DB100 Process Values" area="DB" dbNumber="100">
  <Elem type="real" byteOffset="0" scaling="0;27648;0;1000" tagCode="FI_101" name="Flow rate" />
  <Elem type="bool" byteOffset="20" bitIndex="0" tagCode="XS_601" name="Pump run" />
  <Elem type="dword" byteOffset="22" tagCode="DO_PACKED" name="Packed DO states" />
</ElemGroup>

<!-- String data -->
<ElemGroup active="true" name="DB200 Alarm messages" area="DB" dbNumber="200">
  <Elem type="string" byteOffset="0" strLen="40" tagCode="MSG_ALARM" name="Active alarm text" />
</ElemGroup>

<!-- Process Inputs -->
<ElemGroup active="true" name="Process Inputs (PE)" area="PE">
  <Elem type="bool" byteOffset="0" bitIndex="0" tagCode="I0_0" name="Input I0.0" />
  <Elem type="word" byteOffset="2" tagCode="IW2" name="Input word IW2" />
</ElemGroup>
```

**Commands (Write Operations)**:
```xml
<Cmds>
  <Cmd name="Flow setpoint" cmdNum="1" cmdCode="SET_FLOW"
       area="DB" dbNumber="100" byteOffset="30" type="real"
       scaling="0;27648;0;10000" />
</Cmds>
```

**Sample Template**: See `DrvSig7_Example.xml` in the repository.

---

## 📥 How to Get the Binaries

This repository contains **documentation and template examples only**.  
The compiled `.dll` files are distributed via the **Releases** tab on this GitHub page.

---

## 🔑 Licensing

| Driver | License Type | Tag Limit |
|--------|--------------|-----------|
| **DrvSigModbus** | Free and open‑source | Unlimited |
| **DrvSigPccc** | Commercial (Trial / Personal / Professional / Enterprise) | Trial: 10 / Personal: 100 / Professional: 500 / Enterprise: Unlimited |
| **DrvSig7** | Commercial (Trial / Personal / Professional / Enterprise) | Trial: 10 / Personal: 100 / Professional: 500 / Enterprise: Unlimited |

### Trial Mode
- No time limit.
- Only the first **10 tags** defined in the device template are active.
- Full functionality for evaluation purposes.

### License Tiers

| Tier | Tag Limit | Target Users |
|------|-----------|--------------|
| **Personal** | 100 tags | Individuals, students, hobbyists, small projects |
| **Professional** | 500 tags | Industrial applications, commercial projects, most production environments |
| **Enterprise** | Unlimited | Large‑scale deployments, system integrators, OEMs |

### Obtaining a License
1. Run the driver and **copy the HWID** from the communication log.
2. Choose your license tier (Personal / Professional / Enterprise).
3. Send the HWID and your chosen tier to **ketut.kumajaya@gmail.com**.
4. Receive the license file (e.g., `DrvSig7.lic`).
5. Place it in the Rapid SCADA `Config` directory.
6. Restart the communication line.

Example log output:
```
This machine HWID: A1B2C3D4E5F67890
DrvSig7: License file not found. Trial mode active (10 tags max).
```

---

## 🙏 Acknowledgments

This project stands on the shoulders of these great open-source projects:

- [Rapid SCADA](https://rapidscada.org/) – The SCADA platform that powers these drivers.
- [Sharp7](https://github.com/davenardella/Sharp7) – S7 communication library (MIT License).
- [PCCCComm](https://github.com/kumajaya/PCCCComm) – PCCC communication library (GPLv3).
- [BouncyCastle.Cryptography](https://www.bouncycastle.org/) – Cryptography library (MIT).
- [DeviceId](https://github.com/MatthewKing/DeviceId) – Hardware ID generation (MIT).

Thank you to all the developers who make open-source industrial automation possible!

---

## 📞 Support & Contact

For technical support, template assistance, or license purchasing:

📧 Email: **ketut.kumajaya@gmail.com**  
🌐 Website: [blog.kiiota.com](https://blog.kiiota.com/)

---

## 📄 License

- **DrvSigModbus** – Free to use. Based on Rapid SCADA's built-in driver; modifications are released under open‑source terms.
- **DrvSigPccc** & **DrvSig7** – **Commercial software**. See the `LICENSE` file for full terms.

Third‑party libraries are governed by their respective licenses.

---

## 🚀 Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.

---

## 🤝 Contributing
Bug reports and feature requests are welcome via [Issues](https://github.com/kumajaya/scada-drivers/issues).

---

**© 2026 Ketut Kumajaya. All rights reserved.**
