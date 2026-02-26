---
tags: [dcfc, hardware, phytec, everest, controller]
created: 2026-02-26
---

# Phytec SBC Replacement for CM5 Main Controller

This note evaluates **Phytec System-on-Modules (SOMs)** as an industrial-grade replacement for the current [[02 - CM5 based Main Controller|Raspberry Pi CM5]] used as the DCFC main controller running Linux + EVerest.

## 1. Why Replace the CM5

The Raspberry Pi CM5 (4GB, $60) paired with a CM5 IO Board + PoE HAT ($50) and 2x Waveshare Iso CAN + RS485 controllers ($70 each) totals **~$250** for compute + CAN connectivity. While functional for prototyping, this setup has significant limitations for industrial DCFC deployment:

| Limitation | Impact |
|---|---|
| **No native CAN bus** | Requires 2x external Ethernet-CAN bridges (Waveshare), adding cost, latency, and failure points |
| **Consumer temperature range** | 0°C to +50°C — insufficient for outdoor DCFC enclosures that see -20°C to +70°C ambient |
| **No real-time cores** | Safety-critical control loops (contactor sequencing, fault response) run on Linux with no deterministic timing guarantee |
| **Uncertain availability** | RPi follows 2–3 year consumer product cycles; industrial chargers need 10–15 year supply |
| **No security subsystem** | No hardware root of trust, no secure boot by default — relevant for ISO 15118 Plug & Charge and Cyber Resilience Act compliance |

> [!note] Phytec is already a supplier
> Phytec PLC Controllers (2x $200) are already in the [[02 - CM5 based Main Controller|current BOM]]. Consolidating on a single vendor for both the main controller and PLC modems simplifies procurement and support.

## 2. Phytec Platform Overview

**PHYTEC** is a German embedded systems company (Mainz, founded 1986) specializing in industrial System-on-Modules with **15+ year product availability guarantees**. They are a TI Design Network partner and manufacture in Germany.

### EV Charging Focus

Phytec has made EV charging a strategic vertical:

- **phyVERSO-EVCS** — A 2-port EV charging controller pre-configured with the Pionix EVerest/BaseCamp charger OS, supporting V2G, PLC, OCPP, with Ethernet/WLAN/BLE/LTE connectivity and TPM 2.0 security
- **EVCS-Cube** — Evaluation platforms (Basic for AC, PoC for DC) integrating the phyVERSO controller with EVerest stack for rapid prototyping
- **EVSE Expansion Board** — Dedicated expansion board with HomePlug Green PHY (Lumissil IS32CG5317) for ISO 15118 PLC communication via SPI
- **Pionix partnership** — Adapted Yocto BSP with integrated EVerest stack, ISO 15118, Autocharge, and RFID support out of the box

> [!tip] Key advantage
> Phytec is the only SOM vendor with **official EVerest BSP integration documentation** and a dedicated EV charging product line, eliminating the need to build a custom software stack from scratch.

## 3. Top 3 Recommended SOMs

### 3.1 phyFLEX-AM62Lx — Purpose-Built for EV Charging

| Attribute | Specification |
|---|---|
| **Processor** | TI AM62Lx — 1–2x Arm Cortex-A53 @ 1.25 GHz |
| **Real-time** | None (no R5F cores) |
| **RAM** | 512 MB – 4 GB DDR4 |
| **Storage** | 4 GB – 256 GB eMMC, OSPI NOR |
| **CAN FD** | **3x native CAN FD** |
| **Ethernet** | 2x GbE (1x on-board PHY, 1x RGMII with TSN/AVB) |
| **USB** | 2x USB 2.0 dual-role |
| **Display** | MIPI DSI / LVDS |
| **Security** | Secure Boot, TrustZone, AES/SHA2 HW acceleration |
| **Temperature** | **-40°C to +85°C** |
| **Dimensions** | 37 × 38 mm |
| **Power** | ~2.5 W typical |
| **Availability** | 15+ years guaranteed |
| **SOM Price** | From ~€26 (volume) |

**Why consider:** Smallest form factor, lowest power, 3x native CAN FD eliminates external bridges entirely. Explicitly marketed for EV charging applications. Most cost-effective option.

**Trade-off:** No real-time cores — safety-critical timing must rely on Linux RT patches or external safety controller (which we already have via the STM32F safety controller).

### 3.2 phyCORE-AM64x — Real-Time Industrial Powerhouse

| Attribute | Specification |
|---|---|
| **Processor** | TI AM64x — 2x Arm Cortex-A53 @ 1.0 GHz |
| **Real-time** | **4x Cortex-R5F** + 1x Cortex-M4F + **2x PRU-ICSSG** |
| **RAM** | 2 GB DDR4 |
| **Storage** | 32 GB – 128 GB eMMC, 64–256 MB OSPI NOR |
| **CAN FD** | 2x native CAN FD |
| **Ethernet** | 2x GbE + **4x Industrial GbE via PRU-ICSSG** (EtherCAT capable) |
| **USB** | 1x USB 2.0 + 1x USB 3.1 dual-role |
| **Display** | None (headless design) |
| **Security** | Secure Boot, TrustZone, crypto accelerators |
| **Temperature** | **-40°C to +85°C** |
| **Dimensions** | 50 × 37 mm |
| **Power** | 2.55 W typical / 3.95 W max |
| **Availability** | 15+ years guaranteed |
| **Dev Kit Price** | $299 |

**Why consider:** The 4x R5F real-time cores can run safety-critical control loops (contactor sequencing, fault response <1 ms) independently of Linux. PRU-ICSSG enables EtherCAT and other industrial Ethernet protocols. Could potentially **replace the separate STM32F safety controller** by running safety firmware on the R5F cores.

**Trade-off:** Only 2x CAN FD (need 3 for power bricks + EVSE AUX + spare). No GPU/display output — requires external HMI solution. Higher complexity due to heterogeneous multicore architecture.

### 3.3 phyCORE-AM62x — Official EVerest Reference Platform

| Attribute | Specification |
|---|---|
| **Processor** | TI AM625 — 4x Arm Cortex-A53 @ 1.25 GHz + 1x Cortex-M4F |
| **Real-time** | 1x Cortex-M4F (limited) |
| **RAM** | 2 GB – 4 GB DDR4 |
| **Storage** | 32 GB – 128 GB eMMC, 64–256 MB OSPI NOR |
| **CAN FD** | **3x native CAN FD** |
| **Ethernet** | 2x GbE (1x on-board PHY, 1x RGMII) |
| **USB** | 2x USB 2.0 dual-role |
| **Display** | **3D GPU** (OpenGL 3.x, Vulkan 1.2), OLDI/LVDS (2x), Parallel 24bpp |
| **Security** | Secure Boot, TrustZone, crypto accelerators |
| **Temperature** | **-40°C to +85°C** |
| **Dimensions** | 43 × 32 mm |
| **Power** | 1.48 W typical / 2.63 W max |
| **Availability** | 15+ years guaranteed |
| **Dev Kit Price** | $199 |

**Why consider:** This is **Phytec's official EVerest reference platform** with published integration documentation, a ready-made Yocto BSP with EVerest stack, and the EVSE Expansion Board for PLC. Quad A53 cores provide the most Linux application performance. 3D GPU enables direct HMI rendering. 3x CAN FD covers all bus requirements. Lowest power draw.

**Trade-off:** M4F core is limited compared to R5F — less suitable for complex real-time safety logic. Slightly less industrial protocol support than AM64x.

## 4. Comparison Table — CM5 vs Phytec Options

| Dimension | RPi CM5 (Current) | phyFLEX-AM62Lx | phyCORE-AM64x | phyCORE-AM62x |
|---|---|---|---|---|
| **Application Cores** | 4x A76 @ 2.4 GHz | 1–2x A53 @ 1.25 GHz | 2x A53 @ 1.0 GHz | 4x A53 @ 1.25 GHz |
| **Real-Time Cores** | None | None | 4x R5F + M4F + 2x PRU | 1x M4F |
| **Native CAN FD** | **0** | **3** | **2** | **3** |
| **Ethernet** | 1x GbE | 2x GbE (TSN) | 2x GbE + 4x Industrial | 2x GbE |
| **GPU / Display** | VideoCore VII | MIPI DSI/LVDS | None (headless) | 3D GPU, OLDI/LVDS |
| **RAM** | 4 GB LPDDR4X | Up to 4 GB DDR4 | 2 GB DDR4 | Up to 4 GB DDR4 |
| **Temp Range** | 0°C to +50°C | -40°C to +85°C | -40°C to +85°C | -40°C to +85°C |
| **Secure Boot** | Limited | Yes (TrustZone) | Yes (TrustZone) | Yes (TrustZone) |
| **EVerest BSP** | Manual setup | Community | Community | **Official docs + BSP** |
| **Availability** | 2–3 year cycles | 15+ years | 15+ years | 15+ years |
| **SOM Price (est.)** | ~$60 | ~€26+ (volume) | ~€40+ (volume) | ~€35+ (volume) |
| **Total System Cost** | ~$250 (w/ CAN bridges) | ~$80–120 (w/ carrier) | ~$120–160 (w/ carrier) | ~$100–140 (w/ carrier) |

## 5. Impact on BOM

Switching to a Phytec SOM with native CAN FD eliminates several components:

| Component | Current Cost | After Migration | Change |
|---|---|---|---|
| Raspberry Pi CM5 (4 GB) | $60 | — | **Remove** |
| CM5 IO Board + PoE HAT | $50 | — | **Remove** |
| Waveshare Iso CAN + RS485 (x2) | $140 | — | **Remove** |
| Phytec SOM (AM62x or AM62Lx) | — | ~$35–50 | **Add** |
| Phytec Carrier Board (custom or dev) | — | ~$60–80 | **Add** |
| EVSE Expansion Board (PLC) | — | ~$50 (est.) | **Add** (replaces part of PLC cost) |
| **Subtotal (compute + CAN)** | **$250** | **~$145–180** | **Save $70–105** |

> [!note] Additional savings potential
> - The phyCORE-AM64x with R5F cores could potentially **eliminate the $120 STM32F safety controller** if safety firmware is ported to the R5F cores, saving an additional $120
> - Consolidating on Phytec for both main controller and PLC modems may unlock volume pricing on the existing 2x PLC Controllers ($400 total)
> - Reduced cabling and fewer board-to-board connections improve reliability

## 6. Recommendation

### Primary Pick: phyCORE-AM62x

The **phyCORE-AM62x** is the recommended choice for the following reasons:

1. **Official EVerest integration** — Phytec publishes Yocto BSP documentation with EVerest pre-integrated, dramatically reducing software bring-up time. No other SOM vendor offers this
2. **3x native CAN FD** — Covers all three bus requirements (power bricks, EVSE AUX, spare/diagnostics) without external bridges
3. **Quad A53 + GPU** — Strongest Linux application performance of the three, plus direct HMI rendering eliminates need for a separate display controller
4. **Lowest power** — 1.48 W typical simplifies thermal design in sealed DCFC enclosures
5. **EVSE Expansion Board** — Ready-made HomePlug Green PHY board for ISO 15118 PLC, validated with EVerest
6. **Ecosystem alignment** — Same vendor as existing PLC controllers; consolidates supply chain
7. **$199 dev kit** — Lowest barrier to evaluation

### Secondary Pick: phyCORE-AM64x

Consider the AM64x **only if** the project later requires:
- Eliminating the separate STM32F safety controller (R5F cores can take over)
- EtherCAT or other industrial Ethernet protocols (PRU-ICSSG)
- Headless operation where HMI is handled by a separate panel

## 7. Next Steps

- [ ] **Order phyCORE-AM62x Development Kit** ($199) from [phytec.com](https://www.phytec.com/product/phycore-am62x/) — includes SOM + phyBOARD carrier
- [ ] **Order EVSE Expansion Board** from [phytec.com](https://www.phytec.com/product/evse-expansion-board/) for PLC evaluation
- [ ] **Flash Yocto BSP with EVerest** using [Phytec's EV Charging docs](https://docs.phytec.com/projects/yocto-phycore-am62x/en/latest/3rdpartyintegration/ev-charging.html)
- [ ] **Test CAN FD interfaces** — verify communication with DC power brick simulator and EVSE AUX board prototype
- [ ] **Benchmark EVerest startup time** — compare cold boot to operational on AM62x vs. CM5
- [ ] **Evaluate carrier board requirements** — determine if phyBOARD-Lyra suffices or if a custom carrier is needed for the DCFC form factor
- [ ] **Contact PHYTEC sales** (dennis.hering@phytec.de) for volume SOM pricing and custom carrier board design support
- [ ] **Request EVCS-Cube PoC** for DC charging evaluation if budget permits

## Sources

- [phyFLEX-AM62Lx FPSC — PHYTEC](https://www.phytec.eu/en/produkte/system-on-modules/phyflex-am62lx-fpsc/)
- [phyCORE-AM64x SOM — PHYTEC](https://www.phytec.com/product/phycore-am64x/)
- [phyCORE-AM62x SOM — PHYTEC](https://www.phytec.com/product/phycore-am62x/)
- [EV Charging — phyCORE-AM62x Docs](https://docs.phytec.com/projects/yocto-phycore-am62x/en/latest/3rdpartyintegration/ev-charging.html)
- [phyVERSO-EVCS Charging Controller — PHYTEC](https://www.phytec.eu/en/ladeelektronik/komplettloesung/)
- [EV Charging Electronics — PHYTEC](https://www.phytec.eu/en/produkte/ladeelektronik/)
- [EVSE Expansion Board — PHYTEC](https://www.phytec.com/product/evse-expansion-board/)
