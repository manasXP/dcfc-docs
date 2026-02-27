# DC Fast Charger — Technical Investor Presentation

Tags: #dcfc #presentation #investor

---

## Slide 1: Title

### 150 kW DC Fast Charger
**Modular, Open-Source, SiC-Based EV Charging Infrastructure**

- In-house power electronics design
- EVerest open-source software framework (Linux Foundation Energy)
- 400V and 800V EV compatibility
- Scalable 50 kW — 350 kW architecture

---

## Slide 2: Market Opportunity

### Global DCFC Market Context

| Metric | Value |
|--------|-------|
| Global EV sales (2025) | ~17 million units |
| DC fast chargers needed globally by 2030 | ~2.4 million (IEA estimate) |
| Average commercial DCFC price (150 kW) | $30,000 — $60,000 |
| Our target BOM cost (150 kW, @100 units) | **$21,500** |
| Our target BOM cost (@500 units) | **$19,486** |

**Key insight:** Most DCFC manufacturers use proprietary, vertically integrated software stacks. Our open-source EVerest approach eliminates software licensing costs and vendor lock-in while accelerating time-to-market.

---

## Slide 3: Product Overview

### 150 kW CCS2 DC Fast Charger — System Summary

| Parameter | Specification |
|-----------|---------------|
| AC Input | 3-phase 400–480 Vac, 50/60 Hz |
| DC Output Voltage | 150–1000 VDC |
| DC Output Current | Up to 500 A |
| Maximum Power | 150 kW (5× 30 kW modules) |
| Peak Efficiency | >96% (SiC-based) |
| Power Factor | >0.99 |
| Connector | CCS Combo 2 (CCS1 optional) |
| Charging Time (80%) | 400V battery: 20–30 min · 800V battery: 15–20 min |
| Software | EVerest (Linux Foundation Energy, open-source) |
| Cooling | Sealed cabinet + clip-on HVAC unit |
| Enclosure | IP55 target |

---

## Slide 4: System Architecture

### 4-Level Hierarchical Control

```
LEVEL 3: CLOUD/BACKEND
  ┌──────────────────────────────────────────┐
  │  CSMS (OCPP 2.0.1)  │  Fleet Management  │
  └──────────────┬───────────────────────────┘
                 │ WebSocket / 4G / Ethernet
LEVEL 2: STATION CONTROLLER
  ┌──────────────┴───────────────────────────┐
  │  Phytec phyCORE-AM62x Main Controller (Linux + EVerest)   │
  │  EvseManager · EnergyMgr · OCPP · Auth   │
  │  ISO 15118 · Safety BSP · Power Driver   │
  └──┬────────────────┬─────────────────┬────┘
     │ CAN #2         │ CAN #1 (FD)     │ CAN #3
LEVEL 1: MODULE CONTROLLERS
  ┌──┴──┐    ┌────────┴────────┐    ┌───┴───┐
  │Safety│   │ PDU-Micro Master│    │ HVAC  │
  │Supv. │   │ (Module 0 PIM)  │    │ Unit  │
  │STM32 │   └───┬──┬──┬──┬────┘    └───────┘
  └──────┘       M1  M2  M3  M4

LEVEL 0: HARDWARE (Gate Drivers, Sensors, Contactors)
```

- **Separation of concerns:** Safety-critical functions on dedicated STM32 (SIL 2), not on Linux
- **Three independent CAN buses** for isolation between safety, power, and thermal domains
- **Fail-safe by design:** Hardware interlock chain operates independently of software

---

## Slide 5: Power Conversion — Modular Architecture

### 5× 30 kW Hot-Swappable Power Modules = 150 kW

```
┌─────────────────────────────────────────────────┐
│              POWER SHELF (150 kW)               │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐    │
│  │ Mod 1  │ │ Mod 2  │ │ Mod 3  │ │ Mod 4  │    │
│  │ 30 kW  │ │ 30 kW  │ │ 30 kW  │ │ 30 kW  │    │
│  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘    │
│      └─────┬────┴─────┬────┴─────┬────┘         │
│            │    120 kW + Mod 5 (30 kW)          │
│  ┌────────┐│                                    │
│  │ Mod 5  ││          OUTPUT BUS (DC)           │
│  │ 30 kW  │├────────> 150–1000 VDC, 0–500 A     │
│  └────────┘│          Total: 150 kW             │
└────────────┴────────────────────────────────────┘
```

**Scalability:**
| Configuration | Modules | Power | BOM @100 |
|---------------|---------|-------|----------|
| Single CCS2 | 5× 30 kW | 150 kW | $21,503 |
| Dual CCS2 | 5× 30 kW | 150 kW shared | ~$25,700 |
| Single CCS2 | 10× 30 kW | 300 kW | ~$32,900 |
| Dual CCS2 | 12× 30 kW | 350 kW | ~$36,900 |

**Benefits:** Hot-swappable · N+1 redundancy · Graceful degradation · Field-scalable power

---

## Slide 6: Power Module — PDU-Micro (30 kW)

### In-House Designed SiC Power Converter

| Parameter | Specification |
|-----------|---------------|
| Rated Power | 30 kW continuous |
| PFC Topology | Vienna Rectifier, 140 kHz switching |
| DC-DC Topology | 3-phase Interleaved Dual Active Bridge (DAB) |
| Switching Devices | SiC MOSFETs (Microchip MSCSM120VR1M16CTPAG PFC module; MSC080SMA120B DAB discrete) |
| Output Voltage | 150–1000 VDC |
| Output Current | 100 A per module |
| DC Bus | 700–920 VDC (adaptive) |
| Controller | Dual dsPIC33CH512MP506 (PFC PIM + DAB PIM) |
| Communication | CAN-FD 500k/2M bps |
| Peak Efficiency | 96.3% (system-level) |
| Current Status | Rev 0.7 PCB, firmware in active development |

**Top cost drivers per module:** DAB SiC MOSFETs (23%), Vienna PFC module (13.3%), PCB fabrication (13%)

**Unit cost:** $1,806 @100 units · $1,436 @500 units · **$60/kW @100 · $48/kW @500**

---

## Slide 7: Safety Architecture

### Dual-Layer Safety: Hardware Interlock + Software Supervision

```
+24V Safety Supply
    │
    ▼
┌───────────────────────────────────────────────┐
│         HARDWARE INTERLOCK CHAIN              │
│  (Series NC contacts — any break = shutdown)  │
│                                               │
│  E-STOP → Door Interlock → IMD Contact →      │
│  RCD Contact → Thermal Trip → Safety Relay    │
│                                               │
│  Output: AC Contactor · DC Contactor ·        │
│          Pre-charge Relay                     │
└───────────────────────────────────────────────┘

Main controller/EVerest MONITORS but does NOT own the safety loop.
Hardware trips independently even if main controller crashes.
```

**Safety State Machine:** INIT → IDLE → AC_CLOSE → PRECHARGE → DC_CLOSE → CHARGING → SHUTDOWN → FAULT

**Protections:**
| Protection | Response Time | Method |
|------------|---------------|--------|
| Over-Voltage (OVP) | < 1 µs | Hardware comparator |
| Over-Current (OCP) | < 1 ms | Safety supervisor ADC |
| Ground Fault | < 30 ms | IMD relay (Bender isoCHA425HV) |
| Insulation Failure | During CableCheck | IMD measurement |
| Main Controller Heartbeat Loss | 2 s timeout | Orderly shutdown |
| Contactor Weld | Post-open check | Aux NC + voltage verify |

**Target:** IEC 61508 SIL 2 / ISO 13849 PLd

---

## Slide 8: Software Stack — EVerest Framework

### Open-Source Charging OS (Linux Foundation Energy)

```
┌─────────────────────────────────────────────┐
│           APPLICATION LAYER                 │
│  EvseManager · EnergyManager · Auth · OCPP  │
├─────────────────────────────────────────────┤
│           MIDDLEWARE LAYER                  │
│  ISO 15118 · OCPP 2.0.1 · IEC 61851         │
│  Safety State Machine                       │
├─────────────────────────────────────────────┤
│      HARDWARE ABSTRACTION LAYER             │
│  CAN · Ethernet · GPIO · UART · SPI         │
├─────────────────────────────────────────────┤
│       OPERATING SYSTEM (Linux)              │
│       phyCORE-AM62x (4-core A53, 4 GB RAM)            │
└─────────────────────────────────────────────┘
```

**Why EVerest:**
- **No licensing fees** — fully open-source under Apache 2.0
- **Pre-built protocol stacks** — ISO 15118, OCPP, IEC 61851 tested and community-maintained
- **Microservices via MQTT** — modules run as independent Linux processes
- **Active community** — Linux Foundation Energy backing, major OEM contributions
- **Certification-ready** — conformance-tested protocol implementations

**Custom EVerest Modules (our IP):**
1. `SafetySupervisorBSP` — CAN #2 bridge to STM32 safety controller
2. `PowerModuleDriver` — CAN-FD bridge to PDU-Micro module master
3. `HvacDriver` — CAN #3 bridge to HVAC thermal management unit

---

## Slide 9: Communication & Protocol Stack

### Standards-Compliant Vehicle and Network Communication

**Vehicle Side (EV ↔ Charger):**
| Protocol | Standard | Function |
|----------|----------|----------|
| Control Pilot | IEC 61851-1 | Basic signaling (States A–F) |
| PLC (HomePlug GreenPHY) | ISO 15118-3 | Physical layer for HLC |
| DIN SPEC 70121 | DIN 70121 | Legacy DC fast charging |
| ISO 15118-2 | ISO 15118-2 | Plug & Charge, smart charging |

**Network Side (Charger ↔ Backend):**
| Protocol | Version | Function |
|----------|---------|----------|
| OCPP | 2.0.1 | Remote monitoring, smart charging, billing |
| TLS 1.2/1.3 | ISO 15118 | Secure Plug & Charge certificates |
| MQTT | Internal | EVerest inter-module messaging |

**Connectivity:** Ethernet (LAN) + 5G modem (WAN) + WiFi (service) + PoE switch

---

## Slide 10: Thermal Management — Clip-On HVAC

### Sealed Cabinet with External Heat Rejection

```
    HVAC CLIP-ON UNIT              MAIN CABINET
  ┌─────────────────┐           ┌─────────────────┐
  │  Condenser +    │           │  Power Modules  │
  │  Compressor     │           │  M1  M2  M3  M4 │
  │       │         │           │  M5             │
  │  Evaporator     │  Cold Air │  Internal       │
  │  Coil      ═══════════════> │  Heatsinks      │
  │       │         │           │       │         │
  │  Blower Fans    │  Hot Air  │  Hot Air Return │
  │            <═══════════════ │                 │
  │                 │           │                 │
  │  HVAC Controller│◄──CAN #3─►│  Main ECU (AM62x)   │
  └─────────────────┘           └─────────────────┘
```

**Key advantage:** Internal air never mixes with ambient → sealed IP55 enclosure, no dust ingress

| Parameter | Specification |
|-----------|---------------|
| Cooling Capacity | 15–25 kW thermal |
| Airflow | 2,000–4,000 CFM (variable speed) |
| Operating Range | -20°C to +50°C ambient |
| Refrigerant | R-410A or R-32 |
| Mounting | Side, rear, or top clip-on |
| Interface | CAN bus + 24V power |
| Serviceability | Field-replaceable as single unit |

**Derating chain:** If cooling is insufficient → HVAC signals main controller → EnergyManager reduces power → prevents thermal shutdown

---

## Slide 11: Competitive Analysis

### 150 kW Class DC Fast Chargers

| Feature | Our Design | ABB Terra 184 | Tritium PKM150 | Delta UFC 150 | Kempower S-Series |
|---------|-----------|---------------|----------------|---------------|-------------------|
| **Max Power** | 150 kW (modular to 350) | 180 kW | 150 kW | 150 kW | 40–600 kW |
| **Efficiency** | >96% (SiC) | 95% | ~95% | >94% | N/A |
| **DC Output** | 150–1000V | 150–920V | 150–920V | 200–1000V | 50–920V |
| **Max Current** | 500 A | 200 A | 350 A | 500 A | 500 A |
| **IP Rating** | IP55 (target) | IP54 | IP65 | IP55 | IP54 |
| **Software** | EVerest (open) | Proprietary | Proprietary | Proprietary | Proprietary |
| **SiC MOSFETs** | Yes | Not confirmed | Not confirmed | Not confirmed | Not confirmed |
| **OCPP** | 2.0.1 | 1.6J | 1.6J / 2.0.1 | 1.5S / 1.6J | 1.6J / 2.0.1 |
| **Retail Price** | Target: competitive | $40,000–50,000 | $35,000–45,000 | $30,000–45,000 | $35,000–50,000 |

**Our differentiators:**
1. Open-source software — no vendor lock-in, no licensing fees
2. SiC throughout — highest efficiency in class
3. Full 1000V output — future-proofed for 800V EVs
4. Modular power — same platform scales 50–350 kW
5. In-house power electronics — full control of cost and supply chain

---

## Slide 12: Bill of Materials Breakdown

### 150 kW Single CCS2 — Component Cost @100 Units

```
Power Modules (5×30 kW)      ████████████████████████████  $10,087  (46.9%)
CCS Connector + Cable         ██████████                    $2,180   (10.1%)
Cabinet and Enclosure         ██████████                    $2,145   (10.0%)
DC Output Protection           ████████                     $1,785    (8.3%)
HVAC Clip-On Unit             ████████                     $1,720    (8.0%)
AC Input and Protection       ███████                      $1,496    (7.0%)
Control Electronics           ████                         $730      (3.4%)
Cooling System                ███                          $635      (3.0%)
Networking + Communication    ██                           $375      (1.7%)
User Interface                █                            $200      (0.9%)
Auxiliary Power Supply        █                            $150      (0.7%)
─────────────────────────────────────────────────────────────────────
TOTAL BOM                                                  $21,503
```

| Volume | BOM Cost | Target Retail | Gross Margin |
|--------|----------|---------------|--------------|
| @100 units | $21,503 | $35,000–40,000 | 40–46% |
| @500 units | $19,486 | $30,000–35,000 | 35–44% |

*Excludes: assembly labor, installation, shipping, certification costs, software (open-source), warranty reserve*

---

## Slide 13: Cost Optimization Roadmap

### Path to Further BOM Reduction

| Initiative | Savings per Unit | Timeline |
|------------|-----------------|----------|
| Chinese SiC alternatives (DAB MOSFETs) | $500–$650 | Near-term |
| Volume pricing on COTS components (CCS, contactors) | $1,000–$1,500 | @500+ units |
| PCB panel optimization and assembly volume | $300–$500 | @500+ units |
| Backplane redesign (simplified busbar) | $100–$200 | Rev B |
| In-house HVAC manufacturing | $300–$500 | Year 2 |
| **Projected BOM @1000 units** | | **~$16,500–$17,500** |

**At $17,000 BOM and $32,000 retail: ~47% gross margin**

**Software cost advantage:** Competitors spend $2,000–$5,000 per unit on proprietary software licensing. Our EVerest-based stack has $0 per-unit software cost.

---

## Slide 14: Standards & Certification Plan

### Compliance Matrix

| Standard | Scope | Status |
|----------|-------|--------|
| IEC 61851-1 | General EV charging requirements | Design phase |
| IEC 61851-23 | DC charging station requirements | Design phase |
| IEC 62368-1 | Electrical safety | Testing planned |
| IEC 61000-6-2/6-4 | EMC (emissions + immunity) | Pre-compliance M11 |
| ISO 15118-2/3 | Vehicle-to-grid communication | EVerest pre-certified |
| ISO 13849 (PLd) | Safety of machinery | Safety supervisor design |
| IEC 61508 (SIL 2) | Functional safety | Safety supervisor design |
| OCPP 2.0.1 | Charge point protocol | EVerest pre-certified (libocpp) |
| UL 2202 | US market certification | Planned for US launch |

### Certification Bodies

| Region | Body | Certification |
|--------|------|---------------|
| Europe | TÜV | CE marking, IEC 61851, EMC |
| North America | UL | UL 2202, NEC Article 625 |
| India | BIS | IS 17017 (planned) |

### Certification Timeline: Months 12–14 of development plan

---

## Slide 15: Development Plan — 14-Month Timeline

### 8 Parallel Workstreams

```
Month:  1    2    3    4    5    6    7    8    9   10   11   12   13   14
        ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
WS-1    ████████████████████████████████████████                PDU-Micro Modules
WS-2         ████████                                           CAN Protocol Bridge
WS-3    ████████████████████████                                Safety Supervisor FW
WS-4         ░░░░████████████████████████████████                EVerest Integration
WS-5    ████████████████████████                                Cabinet Hardware
WS-6              ████████████████                              HVAC Controller FW
WS-7                                  ████████████████          System Integration
WS-8                                            ████████████████████  Certification

████ = Active development    ░░░░ = Setup/preparation
```

**Critical Path:** Module Firmware → CAN Bridge → EVerest Modules → System Integration → Certification

---

## Slide 16: Key Milestones

### 14-Month Execution Plan

| Month | Milestone | Gate Criteria |
|-------|-----------|---------------|
| M2 | Safety supervisor board powered | CAN #2 loopback verified |
| M3 | First PDU-Micro module producing DC | 30 kW output on bench |
| M4 | phyCORE-AM62x running EVerest | SafetySupervisorBSP talking to STM32 |
| M5 | PowerModuleDriver controlling 1 module | CAN-FD setpoint control verified |
| M6 | Cabinet assembled, 5 modules installed | Backplane wired, mechanical fit |
| M7 | HVAC clip-on operational | Thermal derating chain verified |
| M8 | **First complete charging session** | ISO 15118 with EV simulator |
| M9 | **Full-power 150 kW sustained test** | Thermal steady-state verified |
| M10 | OCPP backend + HMI complete | Remote monitoring operational |
| M11 | EMC pre-compliance pass | EN 55032 Class B |
| M12–14 | **Certification testing** | IEC 61851, IEC 62368, EMC |

---

## Slide 17: Team & Resource Requirements

### 8–10 Engineers (3–4 Shared from Existing PDU-Micro Team)

| Role | Count | Key Skills |
|------|-------|------------|
| Power Electronics Engineer | 1–2 | PDU-Micro hardware, magnetics, thermal |
| Embedded FW Engineer (Safety) | 1 | STM32, bare-metal C, MISRA C, IEC 61508 |
| Embedded FW Engineer (Controls) | 1–2 | dsPIC33CH, power converter control loops |
| Linux/EVerest Software Engineer | 1–2 | C++, Linux, EVerest, SocketCAN, MQTT |
| Protocol Engineer | 1 | ISO 15118, OCPP 2.0.1, CAN bus, PLC |
| Electrical/Mechanical Engineer | 1 | Cabinet design, busbar, wiring, IP54 |
| HVAC Engineer | 0.5 | Refrigeration, compressor control (part-time) |
| QA / Certification Engineer | 1 | IEC 61851, IEC 62368, EMC, test planning |

**Leverage:** Existing PDU-Micro team provides power electronics and embedded firmware capability. EVerest open-source community provides protocol stack support.

---

## Slide 18: Risk Mitigation

### Top Technical Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| PDU-Micro firmware delayed | Critical | Medium | EVerest SIL simulation (no hardware needed); decouple EVerest work |
| CAN-FD adapter incompatibility | High | Medium | Verify early (M1); PEAK PCAN-USB FD as backup |
| ISO 15118 PLC modem EV compatibility | High | Medium | Test with multiple EV models; DIN 70121 fallback |
| Safety SIL 2 assessment fails | High | Low | Engage safety assessor early (M3); design for SIL 2 from start |
| Thermal management insufficient at 150 kW | Medium | Low | Thermal budget validated; full-power test at M9 |
| OCPP 2.0.1 backend interoperability | Medium | Medium | Test with multiple CSMS vendors; use EVerest certified libocpp |
| Module current sharing instability (5 modules) | Medium | Low | Droop-based sharing (proven method); bench test early |
| Supply chain disruption (SiC components) | Medium | Medium | Dual-source SiC (Microchip + Chinese alternates); 6-month buffer stock |

---

## Slide 19: Deployment Strategy

### Phase 1 → Phase 3 Market Rollout

**Phase 1 — Pilot Deployment (Months 14–18)**
- 5–10 units deployed at controlled sites (fleet depots, partner locations)
- 6-month field validation period
- Collect reliability data, MTBF metrics, user feedback
- OTA firmware updates via OCPP

**Phase 2 — Regional Launch (Months 18–24)**
- 50–100 units
- Target: highway corridors, urban charging hubs, fleet charging
- OCPP backend integration with CPO partners
- Dual-connector (CCS2 + CCS2) variant

**Phase 3 — Scale Production (Year 2+)**
- 500+ units/year
- 300 kW and 350 kW variants
- Multi-connector power-sharing
- International certifications (UL for US, BIS for India)
- In-house HVAC manufacturing

**Revenue model:** Hardware sales + OCPP backend SaaS (optional) + maintenance contracts

---

## Slide 20: Investment Summary

### Why Invest Now

**Technical Readiness:**
- In-house 30 kW SiC power module at Rev 0.7 — core IP developed
- EVerest framework proven in production chargers globally
- Safety architecture designed to SIL 2 from inception
- Full documentation package: system architecture, hardware, software, CAN protocols

**Cost Advantage:**
- BOM: **$21,500 @100** → **$19,500 @500** → projected **$17,000 @1,000**
- $0 per-unit software licensing (vs. $2,000–$5,000 for competitors)
- Target gross margin: **40–47%** at market-competitive pricing

**Market Position:**
- Only open-source software DCFC with full in-house power electronics
- Highest efficiency in class (>96% SiC-based)
- Full 1000V output — ready for 800V EV wave
- Modular platform: one design covers 50–350 kW

**Ask:**
- 14-month development to certification-ready prototype
- 8–10 person engineering team
- Pilot deployment at Month 14

---

*Document generated from DCFC technical documentation. All specifications, costs, and timelines based on current design status as of February 2026.*

*Contact: [Insert contact information]*

---

## Appendix A: Charging Session Flow (ISO 15118)

```
EV                          Charger (EVerest)                Backend (OCPP)
│                                │                               │
│── SLAC (PLC link setup) ──────>│                               │
│<──────── SLAC Response ────────│                               │
│                                │                               │
│── SDP (Service Discovery) ────>│                               │
│<──────── SDP Response ─────────│                               │
│                                │                               │
│── Session Setup ──────────────>│                               │
│<──────── Session Confirmed ────│── TransactionEvent ──────────>│
│                                │<───── RequestStartTransaction─│
│── Cable Check ────────────────>│                               │
│<──────── IMD OK ───────────────│                               │
│                                │                               │
│── Pre-Charge ─────────────────>│                               │
│<──────── Voltage Matched ──────│                               │
│                                │                               │
│── Current Demand (loop) ──────>│── MeterValues ──────────────> │
│<──────── Current Delivered ────│                               │
│                                │                               │
│── Power Delivery Stop ────────>│── TransactionEvent (end) ───> │
│<──────── Welding Detection ────│                               │
│── Session Stop ───────────────>│                               │
```

## Appendix B: Detailed BOM per Configuration

| Configuration | Change from Base | @100 (USD) | @500 (USD) |
|---------------|-----------------|-----------|-----------|
| 150 kW, single CCS2 | Base | **$21,503** | **$19,486** |
| 150 kW, dual CCS2 | +1 cable, +1 contactor set, +1 PLC | ~$25,700 | ~$23,700 |
| 300 kW, single CCS2 | +5 modules, larger breakers, larger HVAC | ~$32,900 | ~$28,900 |
| 350 kW, dual CCS2 | Full configuration | ~$36,900 | ~$32,900 |
