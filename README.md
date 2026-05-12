# Mini PC — Raspberry Pi Compute Module 5 Carrier Board | KiCad PCB Design

A KiCad-designed **custom carrier board** for the **Raspberry Pi Compute Module 5 (CM5)**, forming a fully functional **Linux-capable Mini PC**. The board interfaces directly with the CM5 module and exposes USB-C, HDMI video output, and microSD Linux boot, powered by a dedicated battery charging and regulation architecture — all designed as a compact multi-sheet hierarchical KiCad project.

You will find the following:
- **Board Schematic:** Hierarchical multi-sheet schematic covering the CM5 interface, HDMI output, USB-C connectivity, microSD boot, battery management, and voltage regulation.
- **PCB Layout:** The physical design of the printed circuit board with component placement and keep-out zones.
- **PCB Routing:** Signal and power routing across the two-layer board following CM5 hardware design guidelines.

---

### Tools Used

The electronic design and development were done using **KiCad**, an open-source EDA software.
The version used for this project is **KiCad 9.0.7**.

#### Footprints used for LEDs:
`0603_1608Metric_Pad1.08x0.95mm_HandSolder`

#### Footprints used for Resistors, Capacitors, Inductors:
`0402_1005Metric_Pad1.22x1.35mm_HandSolder`

All the files are attached in the repository folder.

A custom library was created to import certain symbols into the schematic along with their corresponding footprints and 3D models.

The name of the library is:
- **My library.pretty** : for footprints

---

## System Architecture

The design is organised into **8 hierarchical KiCad sub-sheets**:

| Sub-sheet | File | Function |
|---|---|---|
| RPI-CM5 | `rpicm5.kicad_sch` | CM5108064 module — all peripheral signal routing |
| HDMI | `hdmi.kicad_sch` | HDMI Type-A connector + DDC pull-ups + HPD |
| USB-C | `usbc.kicad_sch` | USB-C receptacle (16-pin, USB 2.0) + CC resistors |
| SD-CARD | `sd.kicad_sch` | MicroSD card slot — 4-bit SDIO boot interface |
| Power Management | `pm.kicad_sch` | MCP73831 Li-Po charger + charging status LED |
| Buck | `buck.kicad_sch` | TPS63001 buck-boost — stable 3.3V from battery input |
| LDO | `ldo.kicad_sch` | LM1117LD-1.8 LDO — fixed 1.8V from regulated input |
| Battery Connector | `bat.kicad_sch` | 2-pin Li-Po battery input connector (J1) |

---

## CM5 Module Interface

The **Raspberry Pi CM5 (CM5108064)** is the central compute element, split across two symbol units (**U1A** and **U1B**) in the schematic to manage its large pin count:

| Interface | CM5 Pins | Carrier Function |
|---|---|---|
| +5V Supply (Input) | 77, 98, 135 | Carrier 5V power rail into CM5 |
| CM5_3.3V (Output) | 84, 86 | CM5-generated 3.3V — internal use |
| CM5_1.8V (Output) | 88, 90 | CM5-generated 1.8V — feeds `1.8v` net on carrier |
| HDMI0 Clock | HDMI0_CLK_P/N (188/190) | Differential clock to HDMI connector |
| HDMI0 TX0–TX2 | TX0_P/N, TX1_P/N, TX2_P/N | TMDS data lanes to HDMI connector |
| HDMI0 Control | HDMI0_SCL (200), HDMI0_SDA (199), HDMI0_HOTPLUG (153) | DDC I2C + HPD to HDMI connector |
| USB 2.0 | USB_P (105), USB_N (103) | D+/D− routed to USB-C receptacle |
| SD Interface | SD_CLK (57), SD_CMD (62), SD_DAT0–3 (63/67/69/61) | 4-bit SDIO to microSD slot |
| CC Pull-downs | CC1 (94), CC2 (96) | 5.1 kΩ to GND — configures USB-C device mode |

> **Note:** The CM5 contains its own onboard PMIC that generates all internal rails. The carrier only needs to supply a clean **+5V** to the module's power input pins. The CM5_1.8V output is re-used on the `1.8v` net for carrier-level peripherals.

---

## Key ICs & Design Decisions

### HDMI Output — HDMI Type-A (J4)
- Driven directly by CM5's **HDMI0** peripheral — no external HDMI bridge or level-shifter required
- **TMDS differential pairs** (TX0, TX1, TX2, CLK) routed from CM5 U1B directly to the J4 HDMI_A connector
- **DDC I2C** (SCL/SDA) lines pulled up to 5V via **R9 and R10 (4.7 kΩ each)** for EDID communication with the connected display
- **HPD (Hot-Plug Detect)** pulled up to 5V via **R15 (5.1 kΩ)** — signals monitor presence back to the CM5
- Local decoupling on the connector's 5V supply: **C5 (10 µF) + C6 (0.1 µF)** bulk + bypass

### USB-C — USB 2.0 (J2, USB_C_Receptacle_USB2.0_16P)
- 16-pin USB-C receptacle wired for **USB 2.0 only** — D+ and D− sourced from CM5 USB_P/USB_N pins
- **CC1 and CC2** each pulled down to GND via **R3 and R4 (5.1 kΩ)** — identifies the port as a **USB device (UFP)** and signals a standard 5V/900 mA draw to the host charger
- No active PD controller — the 5.1 kΩ CC resistors alone are sufficient for charger recognition and device-mode configuration
- **VBUS** from the USB-C receptacle is fed into the Power Management sheet as the charging source for the MCP73831

### MicroSD Boot Slot — J5 (Micro_SD_Card)
- Wired to CM5 SDIO in **4-bit mode**: CLK, CMD, DAT0, DAT1, DAT2, DAT3
- All data and command lines pulled up to **3.3V via 10 kΩ resistors** (R5, R6, R7, R8, R14) for clean defined idle states
- **DAT3/CD** pin doubles as card-detect — pulled up on-board for card presence signalling
- Slot powered by the **3p3v** rail with local decoupling: **C3 (10 µF) + C4 (0.1 µF)**
- Compatible with all standard Linux ARM64 boot images (Raspberry Pi OS, Ubuntu Server, etc.)

### Power Architecture

| Stage | IC / Component | Role |
|---|---|---|
| Battery Input | J1 (Conn_01x02) | 2-pin Li-Po battery connector — `BatP` net |
| Li-Po Charger | MCP73831T-2ACI_OT (U2) | Single-cell Li-Ion/Li-Po charger, input from USB-C VBUS |
| Charge Current | R11 (2 kΩ) on PROG pin | Sets charge current: `I = 1000 / 2kΩ` = **500 mA** |
| Charge Status LED | D1 (RED) + R12 (560 Ω) | STAT pin drives LED — ON during charging, OFF when complete |
| Charger Decoupling | C7 + C8 (4.7 µF each) | Stabilises MCP73831 VDD and VBAT rails |
| 3.3V Regulation | TPS63001 (U3) — Buck-Boost | Stable 3.3V (`3p3v`) across full Li-Po discharge range |
| 3.3V Inductor | L1 (2.2 µH) | TPS63001 switching inductor (L1 and L2 pins) |
| 3.3V Decoupling | C9 (10 µF) + C10 (0.1 µF) + C11 (10 µF) + C12 (10 µF) | TPS63001 input/output stabilisation |
| 1.8V Regulation | LM1117LD-1.8 (U5) — LDO | Fixed 1.8V (`1p8v`) from regulated input |
| 1.8V Decoupling | C13 (10 µF) + C14 (22 µF) | LDO input and output stabilisation |

> **Key Insight:** The **TPS63001 is a buck-boost converter** — it steps the Li-Po output both up and down, guaranteeing a stable 3.3V even as the battery discharges below 3.3V (down to 2.5V). This eliminates the brown-out region common in simple LDO-only designs.

### MCP73831T — Li-Po Battery Charger
- Accepts USB-C **VBUS** as the charging input source (`VDD` pin)
- Charges a single-cell Li-Ion/Li-Po via the `BatP` net (VBAT pin)
- Charge current programmed by **R11 (2 kΩ)** on the PROG pin: `I_CHG = 1000 / R_PROG ≈ 500 mA`
- **STAT pin** (open-drain, active-LOW) drives **D1 (RED LED)** through **R12 (560 Ω)** — lights during active charge, turns off on completion
- End-of-charge termination handled internally — no external components required

### TPS63001 — Buck-Boost Converter (3.3V)
- Converts `VIN` (Li-Po, 2.5 V–5.5 V input range) to a stable **3.3V** on the `3p3v` net
- **L1 (2.2 µH)** switching inductor connected to the L1 and L2 pins
- **R13 (100 Ω)** in the feedback path trims the output voltage
- EN and PS/SYNC pins wired for always-on continuous conduction mode

### LM1117LD-1.8 — 1.8V LDO
- Fixed **1.8V** LDO providing the `1p8v` net used by the CM5 interface
- Input decoupled with **C13 (10 µF)**, output decoupled with **C14 (22 µF)**

---

## Global Power Nets

| Net | Source | Consumers |
|---|---|---|
| `BatP` | J1 battery connector | MCP73831 VBAT, TPS63001 VIN |
| `VBUS` | USB-C J2 | MCP73831 VDD (charge input) |
| `5V` | Carrier 5V | CM5 +5V supply pins (77, 98, 135) |
| `3p3v` | TPS63001 VOUT | SD card VDD, carrier peripherals |
| `1p8v` / `1.8v` | LM1117LD-1.8 VOUT | CM5_1.8V interface net |
| `VDD` | Shared carrier rail | HDMI connector 5V, SD card VDD |
| `GND` | Common ground | All sub-sheets |

---

## Schematic

![Root Schematic](https://github.com/vamsiarya31/Mini-PC-based-on-RPI-CM-5/blob/main/images/Root_Schematic.png.png)

---

## PCB Layout

### Overall PCB Layout
![PCB Layout](https://github.com/vamsiarya31/Mini-PC-based-on-RPI-CM-5/blob/main/images/PCB_Layout.png)

---

## 3D View

### Top View 
![3D  Top View](https://github.com/vamsiarya31/Mini-PC-based-on-RPI-CM-5/blob/main/images/3D_Model.png)

### Bottom View 
![3D  Bottom View](https://github.com/vamsiarya31/Mini-PC-based-on-RPI-CM-5/blob/main/images/Bottom_Layer.png)

---

## Board Specifications

| Parameter | Value |
|---|---|
| Layers | 2 |
| Compute Module | Raspberry Pi CM5 (CM5108064) |
| Supply Input | USB-C VBUS (5V) / Li-Po battery |
| Battery Charger | MCP73831T-2ACI_OT — 500 mA charge current |
| 3.3V Regulation | TPS63001 buck-boost (2.5 V – 5.5 V input range) |
| 1.8V Regulation | LM1117LD-1.8 LDO |
| Video Output | HDMI Type-A — CM5 HDMI0 peripheral (TMDS + DDC + HPD) |
| USB | USB-C receptacle — USB 2.0 (D+/D−), device mode via CC pull-downs |
| Boot Storage | MicroSD — 4-bit SDIO (CLK, CMD, DAT0–DAT3) |
| OS Support | Raspberry Pi OS, Ubuntu Server, and any ARM64 Linux distro |

---

## Linux Deployment

The carrier board supports standard Raspberry Pi OS and any ARM64-compatible Linux distribution. To boot:

1. Flash a compatible OS image onto a microSD card using **Raspberry Pi Imager** or `dd`.
2. Insert the microSD into the carrier board's boot slot (J5).
3. Connect a USB-C cable (5V/2A minimum) to J2 for power — this simultaneously charges the Li-Po battery via the MCP73831.
4. Connect a display via the **HDMI Type-A** port (J4).
5. The CM5 will automatically boot from the microSD.

For headless deployment, access the system over **SSH** once the board is on a network.

---

## License

[![MIT License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

## Contact

Vamsi Krishna — Vamsi.Krishna@iiitb.ac.in &nbsp;|&nbsp; [GitHub Profile](https://github.com/vamsiarya31)

Project Link: [https://github.com/vamsiarya31/Mini-PC-based-on-RPI-CM-5](https://github.com/vamsiarya31/Mini-PC-based-on-RPI-CM-5)

---

## Acknowledgments

I would like to express my sincere gratitude to my college, **International Institute of Information Technology (IIIT-B)**, for providing the resources and support to complete this project.  
I am especially grateful to my professor, **Dr. Kurian Polachan**, for their invaluable guidance, encouragement, and expertise throughout the development of this work.
