# Rapid SCADA Communication Drivers

![Rapid SCADA](https://img.shields.io/badge/Rapid%20SCADA-6.0-blue)
![License](https://img.shields.io/badge/License-Commercial%20%2B%20Free-yellow)

A collection of communication drivers for [Rapid SCADA](https://rapidscada.org/) v6.0.

---

## Available Drivers

| Driver | Protocol / Devices | Key Feature | Status |
|--------|-------------------|-------------|--------|
| **DrvSigModbus** | Modbus RTU / TCP | Built-in Modbus enhanced with **driver‑level scaling** | ✅ **Free** (Unlimited) |
| **DrvSigPccc** | Allen‑Bradley PCCC (SLC, MicroLogix, PLC-5) via DF1, DF1Master, EtherNet/IP, or CSPv4 | Full read/write with **read‑group aggregation** & **pipelining** | 🔒 **Trial** (max 10 tags) |
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

Optimized for PCCC controllers with **automatic read‑group aggregation** and **pipelining** on TCP‑based transports.

#### Line Options (configured in Communication Line → Custom Options)

| Option | Default | Description |
|--------|---------|-------------|
| `Protocol` | `DF1` | `DF1` (point‑to‑point serial), `DF1Master` (RS‑485 multidrop), `EIP` (EtherNet/IP), or `CSP` (CSPv4) |
| `MaxConcurrentRequests` | `2` (EIP/CSP) / `1` (DF1) | Number of PCCC requests allowed in flight simultaneously on TCP links. Increase to reduce round‑trip latency **if your PLC can handle it** (tested on ML1400 – ~2× speedup in Run mode). |
| `MaxMergeGap` | `8` | How many unused words/bits can be skipped when merging adjacent addresses into one physical read. Larger values reduce the number of requests but waste some bandwidth. |
| `Host` / `CspHost` | `192.168.1.10` | IP address for EIP or CSPv4. For CSPv4, use `CspHost` and `CspPort` (default: `2222`). |
| `LsapControlByte` | `0` | For CSPv4 only – set to `0x05` when communicating via RSLinx. |
| `ComPort`, `BaudRate`, `Parity` | `COM1`, `19200`, `None` | Serial settings for DF1/DF1Master. |

**DF1Master‑specific**:
- `SlaveAddress` (default `1`)
- `Rs485Mode` (0 = RTS, 1 = DTR, 2 = Disabled)
- `EchoSuppression` (default `false`)
- `Rs485AssertDelayMs`, `Rs485DeassertDelayMs` (default `1`, `5` ms)

#### Read‑Group Aggregation (Physical Batching)

- All `<Elem>` elements across **all active `<ReadGroup>` containers** are globally flattened and sorted by physical address.
- Elements from different template groups that sit next to each other in PLC memory are merged into a single PCCC request.
- `MaxMergeGap` controls how many unused slots can be skipped to keep requests few.

#### Address Syntax – Full Support

| File Type | Examples | Notes |
|-----------|----------|-------|
| **Binary (B)** | `B3:0`, `B3:0/5` | Always use `ushort` for word‑level reads to avoid sign‑extension (see below). |
| **Integer (N)** | `N7:0`, `N7:5` | Can be `ushort` or `short` – N‑file is genuinely signed. |
| **Status (S)** | `S:1`, `S2:3` | `S` file is signed; use `short` if values can be negative. |
| **Input (I)** | `I:0`, `I1:2` | Use `ushort` – bit‑mapped words, no sign interpretation. |
| **Output (O)** | `O:0`, `O1:1` | Use `ushort` – same as `I`. |
| **Float (F)** | `F8:0` | 32‑bit IEEE 754; use `float`. |
| **Long (L)** | `L20:0` | 32‑bit signed integer; use `int` or `long`. |
| **String (ST)** | `ST21:0` | SLC‑style ASCII string (84 bytes). Use `string`. |
| **Timer (T)** | `T4:0.PRE`, `T4:0.ACC`, `T4:0/EN`, `T4:0/DN` | Sub‑elements via dot (`.PRE`, `.ACC`) or slash (`/EN`, `/DN`, `/TT`) for status bits. |
| **Counter (C)** | `C5:0.PRE`, `C5:0.ACC`, `C5:0/CU`, `C5:0/DN` | Same as Timer. |
| **Control (R)** | `R6:0.LEN`, `R6:0.POS`, `R6:0/EN`, `R6:0/EU`, `R6:0/ER` | Full support for Control file sub‑elements (LEN/POS) and all status bits (`/EN`, `/EU`, `/EM`, `/ER`, `/UL`, `/IN`, `/FD`). |

**⚠️ Important – Signed vs Unsigned for I/O/B Words**

- **N‑file and S‑file** are genuinely **signed** 16‑bit values – use `short` if you expect negative numbers, otherwise `ushort`.
- **B‑file, I‑file, and O‑file** are **bit‑mapped words**; always use `ushort` – reading them as `short` will incorrectly apply sign‑extension to bit 15, corrupting the value.
- For bit‑level elements (`address="B3:0/5"`), the driver automatically handles extraction, and the `dataType` should be `bool`.

#### Commands (Write) – Restrictions for Timer/Counter/Control

When configuring a write command (`<Command>`), the `dataType` choice is **restricted** to avoid writing past the intended single word and corrupting adjacent sub‑elements:

- **Timer (T), Counter (C), Control (R)** – only `bool`, `ushort`, or `short` are allowed.
- Other files (N, B, I, O, F, L) – any type whose word count fits the file’s element size (e.g., `float`, `int`, `long`) is allowed.

This restriction is enforced at template load time, so you won’t see invalid options in the UI.

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

<ReadGroup active="true" name="Control R6 LEN">
  <Elem name="R6_0_LEN" tagCode="R6_0_LEN" address="R6:0.LEN" dataType="ushort" />
  <Elem name="R6_1_LEN" tagCode="R6_1_LEN" address="R6:1.LEN" dataType="ushort" />
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
