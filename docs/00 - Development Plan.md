# 150 kW DC Fast Charger -- Development Plan

> **Date:** 2026-02-27
> **Status:** Draft
> **Tags:** #dcfc #development-plan #firmware #everest

---

## 1. Scope and Objectives

Build a complete 150 kW CCS2 DC fast charger from the following in-house and open-source components:

| Component | Source | Status |
|-----------|--------|--------|
| 30 kW Power Module (PDU-Micro) | In-house design | Rev 0.7, firmware in development |
| Cabinet, backplane, contactors | In-house design | Documentation complete, hardware pending |
| Safety Supervisor (STM32) | In-house firmware | Specification complete, firmware pending |
| HVAC Clip-On Unit | In-house design | Specification complete, hardware pending |
| Main Controller (CM5 + EVerest) | CM5 hardware + open-source EVerest | Research complete, integration pending |
| CCS2 Connector + liquid-cooled cable | Commercial COTS | Specification complete |

**System Configuration:** 5x PDU-Micro modules (30 kW each) = 150 kW, with N+1 redundancy optional.

---

## 2. Architecture Alignment

### 2.1 PDU-Micro vs. Original DCFC Assumptions

The DCFC documentation was originally written assuming 6x 25 kW modules with LLC DC-DC topology. The in-house PDU-Micro uses a different architecture:

| Parameter | Original DCFC Assumption | PDU-Micro Actual |
|-----------|--------------------------|------------------|
| Module power | 25 kW | 30 kW |
| Module count | 6 | 5 |
| Total power | 150 kW | 150 kW |
| PFC topology | Vienna rectifier, 65 kHz | Vienna rectifier, 140 kHz |
| DC-DC topology | LLC resonant | 3-phase interleaved DAB |
| SiC package | Discrete TO-247-4 | PFC: MSCSM120VR1M16CTPAG module; DAB: MSC080SMA120B discrete |
| Controller | Single MCU per module | Dual PIM (2x dsPIC33CH512MP506) |
| CAN interface | CAN 2.0A, 500 kbps | CAN-FD, 500k/2M bps |
| Output voltage | 200--1000 VDC | 150--1000 VDC |
| Output current | 62.5 A/module | 100 A/module |
| Module efficiency | Peak 97% | Peak 96.3% (system) |
| DC bus voltage | 800 VDC fixed | 700--920 VDC (adaptive) |

**Action required:** Update DCFC system docs to reflect PDU-Micro specifications. CAN protocol between CM5 and modules must bridge CAN-FD (PDU-Micro native) to the CM5's CAN adapter.

### 2.2 CAN Protocol Reconciliation

The PDU-Micro defines its own CAN-FD protocol (`pdu_can_protocol.h`) with:
- Intra-module: PFC_STATUS / DAB_STATUS at 1 kHz
- Inter-module: MODULE_STATUS / MASTER_CMD / MASTER_ANNOUNCE at 100 Hz
- Module master election (lowest Module ID PFC PIM)

The DCFC's CM5 `PowerModuleDriver` EVerest module expects a different protocol on CAN #1:
- Setpoint messages (0x010--0x01F) from CM5
- Status/telemetry/fault messages (0x110--0x3CF) from modules
- CANopen NMT heartbeat (0x700+)

**Resolution options:**
- **Option A (recommended):** The PDU-Micro's PFC PIM master handles inter-module coordination (current sharing, stacking) autonomously. CM5 sends high-level setpoints (voltage, total current limit, enable/disable) to the elected module master only. The master distributes to slaves. This leverages the existing PDU-Micro droop-based current sharing.
- **Option B:** CM5 addresses each module individually (as in original DCFC spec). Requires disabling PDU-Micro's internal master election and current distribution. More CM5 software complexity.

### 2.3 Revised System Block Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  CM5 Main Controller (Linux + EVerest)                      │
│  ┌──────────┐ ┌──────────┐ ┌──────┐ ┌──────┐ ┌──────────┐  │
│  │EvseManager│ │EnergyMgr │ │OCPP  │ │Auth  │ │EvseV2G   │  │
│  └────┬─────┘ └────┬─────┘ └──┬───┘ └──┬───┘ └────┬─────┘  │
│       │             │          │        │          │         │
│  ┌────┴────┐  ┌─────┴─────┐                  ┌────┴────┐    │
│  │SafetyBSP│  │PowerModule│                  │EvseSlac │    │
│  │ Driver  │  │  Driver   │                  └─────────┘    │
│  └────┬────┘  └─────┬─────┘  ┌──────────┐                  │
│       │              │        │HvacDriver│                  │
│       │              │        └────┬─────┘                  │
└───────┼──────────────┼─────────────┼────────────────────────┘
        │              │             │
   CAN #2 (500k)  CAN #1 (CAN-FD)  CAN #3 (250k)
        │              │             │
   ┌────┴────┐    ┌────┴────┐   ┌───┴───┐
   │ Safety  │    │PDU-Micro│   │ HVAC  │
   │  Supv.  │    │ Master  │   │ Unit  │
   │ (STM32) │    │(Module 0│   │(STM32/│
   └─────────┘    │ PFC PIM)│   │RP2350)│
                  └────┬────┘   └───────┘
                       │
              CAN-FD internal bus
           ┌─────┬─────┬─────┐
           │     │     │     │
          M1    M2    M3    M4
        (30kW)(30kW)(30kW)(30kW)
```

---

## 3. Work Breakdown Structure

The project is organized into 8 workstreams that run partially in parallel.

### WS-1: PDU-Micro Module Completion
> **Owner:** Firmware + Hardware team
> **Dependency:** PDU-Micro project (separate workspace)
> **Duration:** Ongoing (see PDU-Micro sprint plan SP-01 through SP-27)

This workstream is managed in the [[PDU-Micro]] workspace. Key deliverables needed for DCFC integration:

| # | Task | Priority | PDU-Micro Epic |
|---|------|----------|----------------|
| 1.1 | PFC firmware: Vienna current control, PLL, soft-start, fault handling | Critical | EP-03 |
| 1.2 | DAB firmware: Phase-shift control, CC/CV loops, ZVS management | Critical | EP-03 |
| 1.3 | Intra-module CAN-FD: PFC <-> DAB status exchange at 1 kHz | Critical | EP-03 |
| 1.4 | Inter-module CAN-FD: Master election, current sharing, stacking | Critical | EP-04 |
| 1.5 | Thermal management: NTC monitoring, fan control, derating | High | EP-03 |
| 1.6 | Protection: OVP/OCP/OTP, DESAT, ground fault, recovery logic | Critical | EP-03 |
| 1.7 | Rev A PCB build and board-level bring-up (PFC, DAB, AC-Entry) | Critical | EP-02 |
| 1.8 | System integration: 5-module stacking test on bench | Critical | EP-04 |
| 1.9 | Light-load pulse-skipping and module shedding | Medium | EP-06 |
| 1.10 | DPS/TPS modulation (advanced DAB efficiency improvement) | Low | EP-06 |

**Interface to DCFC:** The module master's CAN-FD interface must support the following commands from CM5:
- Set target voltage (0--1000 V)
- Set total current limit (0--500 A)
- Enable / Disable / Emergency stop
- Query aggregate status (V_out, I_out, P_out, state, faults, temperatures)

### WS-2: CAN Protocol Bridge Layer
> **Owner:** Firmware team
> **Duration:** 4 weeks
> **Dependencies:** WS-1 (CAN-FD protocol defined)

| # | Task | Priority | Notes |
|---|------|----------|-------|
| 2.1 | Define CM5-to-ModuleMaster CAN message set | Critical | Simplify DCFC CAN #1 spec to high-level setpoints only (Option A) |
| 2.2 | Implement CM5-side CAN-FD adapter configuration | High | Waveshare isolated CAN module must support CAN-FD; verify or replace with compatible adapter |
| 2.3 | Add CM5-facing CAN responder in PDU-Micro PFC PIM master firmware | Critical | New CAN message handler alongside existing inter-module protocol |
| 2.4 | Define shared `dcfc_can_protocol.h` header | High | Message IDs, signal definitions, scaling, DLC for both CM5 driver and module master |
| 2.5 | Bench test: CM5 SocketCAN <-> Module Master CAN-FD loopback | High | Verify round-trip latency < 5 ms |

### WS-3: Safety Supervisor Firmware (STM32)
> **Owner:** Firmware team (safety-critical, MISRA C)
> **Duration:** 12 weeks
> **Dependencies:** Cabinet hardware (WS-5) for I/O wiring

| # | Task | Priority | Notes |
|---|------|----------|-------|
| 3.1 | Select STM32 variant and design custom board | Critical | STM32F4xx or STM32G4xx; needs 3x CAN, 12+ ADC, 8+ GPIO, SIL 2 capable |
| 3.2 | Bare-metal HAL bring-up: clocks, GPIO, CAN, ADC, timers | Critical | No RTOS; deterministic 1 ms scan loop |
| 3.3 | Hardware interlock chain integration (E-STOP, door, IMD relay, RCD, thermal) | Critical | Wire safety loop through NC contacts in series |
| 3.4 | Safety state machine: INIT -> IDLE -> AC_CLOSE -> PRECHARGE -> DC_CLOSE -> CHARGING -> SHUTDOWN -> FAULT | Critical | 8 states per spec |
| 3.5 | Contactor sequencing: AC contactor -> pre-charge (K4+R1) -> main DC (K2/K3) -> K4 open | Critical | dV/dt monitoring during pre-charge; 3 s timeout |
| 3.6 | Independent OVP/OCP via dedicated ADC (not trusting CM5 or module values) | Critical | Hardware comparator for < 1 us OVP |
| 3.7 | CM5 heartbeat monitoring: CAN #2, 500 ms period, 2 s timeout | Critical | Heartbeat loss = orderly shutdown |
| 3.8 | Contactor weld detection: aux NC contact check + voltage check | High | 50 ms + 500 ms post-open checks |
| 3.9 | CP signal generation: +/-12V 1 kHz PWM, state detection (A--F) | Critical | Via dedicated analog frontend |
| 3.10 | CAN #2 protocol: implement all 0x100--0x102 (CM5 -> Supv) and 0x200--0x202 (Supv -> CM5) messages | Critical | Per safety supervisor CAN spec |
| 3.11 | Fault handling: 12 fault codes (F01--F12), response times (< 1 ms to 3 s) | Critical | |
| 3.12 | Insulation monitoring: IMD interface (Bender isoCHA425HV via relay or direct) | High | During CableCheck phase |
| 3.13 | RCD monitoring: Type B RCD trip detection | High | Hardware signal to safety loop |
| 3.14 | Watchdog: internal IWDG + external window watchdog | High | SIL 2 requirement |
| 3.15 | Unit tests: > 90% branch coverage, MISRA C:2012 compliance | Critical | IEC 61508 SIL 2, ISO 13849 PLd |
| 3.16 | FMEA and safety analysis documentation | High | Required for certification |

### WS-4: EVerest Integration on CM5
> **Owner:** Software team
> **Duration:** 16 weeks
> **Dependencies:** WS-2 (CAN protocol), WS-3 (safety supervisor CAN), WS-6 (HVAC CAN)

#### 4A. Platform Setup (weeks 1--3)
| # | Task | Priority |
|---|------|----------|
| 4A.1 | CM5 Linux image: Raspberry Pi OS Lite (64-bit), hardened for field use | Critical |
| 4A.2 | Build EVerest from source (`everest-core`) on CM5, verify MQTT broker | Critical |
| 4A.3 | Configure SocketCAN interfaces: `can0` (CAN #1, power modules), `can1` (CAN #2, safety), `can2` (CAN #3, HVAC) | Critical |
| 4A.4 | Networking: ETH0 LAN, 5G modem WAN, PoE switch config | High |
| 4A.5 | PLC modem bring-up: QCA7005 or Lumissil IS32CG5317 as Linux netdev | High |
| 4A.6 | Read-only rootfs + overlay for field reliability | Medium |

#### 4B. Custom EVerest Modules (weeks 3--12)
| # | Task | Priority | Interface |
|---|------|----------|-----------|
| 4B.1 | `SafetySupervisorBSP` -- CAN #2 bridge to STM32 | Critical | `evse_board_support`, `isolation_monitor`, `ac_rcd`, `connector_lock` |
| 4B.2 | `PowerModuleDriver` -- CAN #1 bridge to PDU-Micro master | Critical | `power_supply_DC`, `powermeter` |
| 4B.3 | `HvacDriver` -- CAN #3 bridge to HVAC unit | High | custom `thermal_management`, `powermeter` |
| 4B.4 | Module shedding logic in `PowerModuleDriver` (partial load efficiency) | Medium | Internal to driver |
| 4B.5 | Current distribution and rebalancing in `PowerModuleDriver` | High | Leverages PDU-Micro master's droop sharing |

#### 4C. Standard EVerest Module Configuration (weeks 8--14)
| # | Task | Priority | Notes |
|---|------|----------|-------|
| 4C.1 | `EvseManager` configuration: `charge_mode: DC`, cable check, pre-charge parameters | Critical | Single EVSE initially |
| 4C.2 | `EnergyManager` configuration: site fuse limit, HVAC derating integration | Critical | Energy tree: Grid -> Charger -> EVSE |
| 4C.3 | `Auth` module: RFID token provider + OCPP validator, `FindFirst` algorithm | High | |
| 4C.4 | `OCPP201` module: backend integration, device model config, security profile 2/3 | High | `libocpp` configuration |
| 4C.5 | `EvseV2G` + `EvseSlac`: ISO 15118 stack over PLC modem | Critical | DIN 70121 + ISO 15118-2 minimum |
| 4C.6 | `EvseSecurity`: certificate management for TLS, PnC | Medium | CSR workflow for production |

#### 4D. System YAML Wiring (weeks 12--14)
| # | Task | Priority |
|---|------|----------|
| 4D.1 | Create `dcfc_150kw.yaml` node configuration wiring all modules | Critical |
| 4D.2 | Create `dcfc_150kw_sim.yaml` with SIL testing stubs (no hardware) | High |
| 4D.3 | Integration test: full EVerest stack with simulated CAN devices | High |

#### 4E. HMI and Diagnostics (weeks 10--16)
| # | Task | Priority |
|---|------|----------|
| 4E.1 | Web-based HMI: charge status, energy delivered, session info, tariff display | High |
| 4E.2 | Touchscreen driver: 14" Waveshare TFT, kiosk browser mode | Medium |
| 4E.3 | LED connector ring driver (WS2812 via GPIO/SPI) | Low |
| 4E.4 | Diagnostic/service web UI: module status, temperatures, fault logs | Medium |
| 4E.5 | Remote logging and OTA firmware update pipeline | Medium |

### WS-5: Cabinet and Backplane Hardware
> **Owner:** Electrical/Mechanical team
> **Duration:** 12 weeks (parallel with firmware)
> **Dependencies:** PDU-Micro module mechanical dimensions finalized

| # | Task | Priority | Notes |
|---|------|----------|-------|
| 5.1 | Cabinet mechanical design: 5 module slots, control section, cable management | Critical | IP54 enclosure |
| 5.2 | Backplane busbar fabrication: 3-phase copper, tin-plated, 30x5 mm | Critical | Per backplane power management spec |
| 5.3 | PDU 1 (Power): 2x 175A breakers, 2x 200A AC contactors, CTs | Critical | |
| 5.4 | PDU 2 (Auxiliary): dual SMPS (24V), DC-DC (12V, 5V), LiFePO4 UPS | High | |
| 5.5 | PDU 3 (Cooling): coolant pump, radiator fans, HVAC interface | High | |
| 5.6 | PDU 4 (Comms/HMI): CM5 mounting, Waveshare CAN adapters, PLC modems, touchscreen | High | |
| 5.7 | DC output contactor assembly: K2/K3 (500A), K4 pre-charge, R1 (100 ohm), R2 (10k ohm discharge) | Critical | |
| 5.8 | Safety supervisor board: custom STM32 PCB or dev board + breakout | Critical | |
| 5.9 | Wiring harness: CAN buses (shielded twisted pair), power, sensor cables | High | |
| 5.10 | Blind-mate connectors per module slot (AC + DC + CAN + sense) | High | |
| 5.11 | Module slide-rail integration and mechanical fit test | Medium | |
| 5.12 | CCS2 connector + liquid-cooled cable assembly procurement and integration | Critical | COTS, 5 m cable, quick-disconnect coolant fittings |
| 5.13 | MID energy meter installation and Modbus/RS485 connection | Medium | |

### WS-6: HVAC Controller Firmware
> **Owner:** Firmware team
> **Duration:** 8 weeks
> **Dependencies:** HVAC hardware unit (WS-5.5)

| # | Task | Priority | Notes |
|---|------|----------|-------|
| 6.1 | Select controller: STM32F103 or RP2350, design/source HVAC control board | High | 6 ADC, 3 PWM, CAN 2.0B |
| 6.2 | Sensor bring-up: 4x NTC (cabinet, condenser, evaporator, ambient), refrigerant pressure | High | |
| 6.3 | Compressor inverter drive: PWM speed control, soft-start, protection | Critical | |
| 6.4 | EEV stepper control: superheat PID (target 6 degC) | High | |
| 6.5 | Fan control: internal blower + external condenser fan PWM | High | |
| 6.6 | Cabinet temperature PID (Kp=10, Ki=0.5, Kd=2) | High | |
| 6.7 | CAN #3 protocol: implement all 7 messages per HVAC CAN spec | Critical | Status, faults, diagnostics, commands, config, heartbeat |
| 6.8 | Autonomous safe mode: maintain last setpoint at 70% on CAN loss | High | 30 min timeout then max cooling |
| 6.9 | Defrost cycle logic | Medium | |
| 6.10 | Cold climate package: PTC heater, crankcase heater control | Medium | |
| 6.11 | Integration test: HVAC unit on CAN #3 with CM5 HvacDriver | High | |

### WS-7: System Integration and Testing
> **Owner:** Full team
> **Duration:** 8 weeks (after WS-1 through WS-6 reach minimum viable)
> **Dependencies:** All workstreams

| # | Task | Priority | Notes |
|---|------|----------|-------|
| 7.1 | Bench integration: CM5 + safety supervisor + 1 PDU-Micro module | Critical | First end-to-end power flow |
| 7.2 | Scale to 5 modules: current sharing, module shedding, fault injection | Critical | |
| 7.3 | Full charging session: ISO 15118 SDP -> session setup -> cable check -> pre-charge -> current demand -> shutdown | Critical | Using EV simulator or real EV |
| 7.4 | Safety validation: E-stop, door interlock, IMD fault, RCD trip, CM5 crash, module fault | Critical | Each must result in safe shutdown |
| 7.5 | Thermal validation: full-load sustained run, HVAC derating chain test | High | |
| 7.6 | OCPP backend integration test: remote start/stop, smart charging profiles, firmware update | High | |
| 7.7 | Dual-connector session test (if applicable) | Medium | Power sharing between connectors |
| 7.8 | EMC pre-compliance: conducted emissions (EN 55032 Class B) | High | |
| 7.9 | Efficiency measurement: per-module and system-level across load range | High | |
| 7.10 | Endurance test: 1000 charge/discharge cycles, connector wear | Medium | |

### WS-8: Certification and Production Preparation
> **Owner:** QA + Project Management
> **Duration:** 12--16 weeks (overlaps with WS-7)
> **Dependencies:** WS-7 complete

| # | Task | Priority | Notes |
|---|------|----------|-------|
| 8.1 | IEC 61851-23 (DC charging) type testing | Critical | |
| 8.2 | IEC 62368-1 (safety) type testing | Critical | |
| 8.3 | IEC 61000-6-2/6-4 (EMC) testing | Critical | |
| 8.4 | ISO 15118-2/3 conformance testing | High | CharIN test system |
| 8.5 | OCPP 2.0.1 compliance testing | High | OCA test tool |
| 8.6 | UL 2202 (US market, if required) | Medium | |
| 8.7 | Safety supervisor SIL 2 / PLd assessment | Critical | IEC 61508, ISO 13849 |
| 8.8 | Production test fixture design | Medium | |
| 8.9 | Manufacturing documentation package | Medium | |

---

## 4. Timeline Overview

```
Month:  1    2    3    4    5    6    7    8    9   10   11   12   13   14
        ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
WS-1    ████████████████████████████████████████░░░░                      PDU-Micro
WS-2         ████████                                                     CAN Bridge
WS-3    ████████████████████████                                          Safety FW
WS-4         ░░░░████████████████████████████████                         EVerest
  4A         ████████                                                     Platform
  4B              ████████████████████                                    Modules
  4C                        ████████████████                              Config
  4D                                  ██████                              Wiring
  4E                             ████████████████                         HMI
WS-5    ████████████████████████                                          Cabinet HW
WS-6              ████████████████                                        HVAC FW
WS-7                                  ████████████████                    Integration
WS-8                                            ████████████████████████  Certification

████ = Active development
░░░░ = Preparation / setup
```

**Critical path:** WS-1 (module firmware) -> WS-2 (CAN bridge) -> WS-4B (EVerest modules) -> WS-7 (integration) -> WS-8 (certification)

**Key milestones:**

| Month | Milestone |
|-------|-----------|
| M2 | Safety supervisor board powered, CAN #2 loopback verified |
| M3 | First PDU-Micro module producing DC output on bench |
| M4 | CM5 running EVerest, SafetySupervisorBSP talking to STM32 |
| M5 | PowerModuleDriver controlling 1 module via CAN-FD |
| M6 | Cabinet assembled, 5 modules installed, backplane wired |
| M7 | HVAC clip-on operational, thermal derating chain verified |
| M8 | First complete ISO 15118 charging session (EV simulator) |
| M9 | Full-power 150 kW sustained charging test |
| M10 | OCPP backend integration, HMI complete |
| M11 | EMC pre-compliance pass |
| M12--14 | Certification testing and production prep |

---

## 5. Firmware Repository Structure

### 5.1 Existing Repositories (PDU-Micro)
- `pdu-pfc-firmware` -- dsPIC33CH PFC PIM firmware (MPLAB X / XC-DSC)
- `pdu-dab-firmware` -- dsPIC33CH DAB PIM firmware (MPLAB X / XC-DSC)
- `pdu-micro-docs` -- PDU-Micro documentation mirror

### 5.2 New Repositories Required

```
dcfc-safety-supervisor/
├── Core/
│   ├── main.c                    # 1 ms scan loop
│   ├── safety_fsm.c/.h           # 8-state safety state machine
│   ├── contactor_ctrl.c/.h       # Sequencing, weld detection
│   ├── precharge.c/.h            # dV/dt monitoring, timeout
│   ├── cp_signal.c/.h            # Control Pilot PWM + state detection
│   ├── adc_monitor.c/.h          # Independent OVP/OCP
│   ├── imd_driver.c/.h           # Insulation monitor interface
│   ├── heartbeat.c/.h            # CM5 watchdog
│   └── fault_handler.c/.h        # 12 fault codes, logging
├── CAN/
│   ├── can_protocol.c/.h         # CAN #2 message encode/decode
│   └── dcfc_safety_can.h         # Shared message definitions
├── HAL/
│   ├── stm32_hal_conf.h
│   ├── gpio.c/.h
│   ├── can.c/.h
│   ├── adc.c/.h
│   ├── timer.c/.h
│   └── wdg.c/.h
├── Tests/
│   └── ...                       # Unity framework, MISRA C checker
├── STM32CubeIDE project files
└── README.md
```

```
dcfc-everest-modules/
├── modules/
│   ├── SafetySupervisorBSP/
│   │   ├── manifest.yaml
│   │   ├── SafetySupervisorBSP.cpp/.hpp
│   │   ├── can_bridge.cpp/.hpp         # CAN #2 SocketCAN wrapper
│   │   ├── safety_can_protocol.hpp     # Shared with STM32 repo
│   │   └── CMakeLists.txt
│   ├── PowerModuleDriver/
│   │   ├── manifest.yaml
│   │   ├── PowerModuleDriver.cpp/.hpp
│   │   ├── can_bridge.cpp/.hpp         # CAN #1 CAN-FD SocketCAN wrapper
│   │   ├── module_pool.cpp/.hpp        # Discovery, health tracking
│   │   ├── current_distributor.cpp/.hpp
│   │   ├── meter_aggregator.cpp/.hpp
│   │   ├── pdu_can_protocol.hpp        # Shared with PDU-Micro firmware
│   │   └── CMakeLists.txt
│   ├── HvacDriver/
│   │   ├── manifest.yaml
│   │   ├── HvacDriver.cpp/.hpp
│   │   ├── can_bridge.cpp/.hpp         # CAN #3 SocketCAN wrapper
│   │   ├── hvac_can_protocol.hpp
│   │   ├── derate_logic.cpp/.hpp
│   │   └── CMakeLists.txt
│   └── interfaces/
│       └── thermal_management.yaml     # Custom EVerest interface
├── configs/
│   ├── dcfc_150kw.yaml                 # Production node configuration
│   ├── dcfc_150kw_sim.yaml             # SIL testing configuration
│   └── component_config/               # OCPP device model
├── scripts/
│   ├── setup_can.sh                    # CAN interface setup
│   └── deploy.sh                       # CM5 deployment
├── tests/
│   └── ...                             # pytest + EVerest test framework
├── CMakeLists.txt
└── README.md
```

```
dcfc-hvac-firmware/
├── Core/
│   ├── main.c
│   ├── thermal_ctrl.c/.h          # PID controllers (cabinet, superheat, condenser)
│   ├── compressor_ctrl.c/.h       # Inverter drive, soft-start
│   ├── eev_ctrl.c/.h              # Stepper motor EEV
│   ├── fan_ctrl.c/.h              # Blower + condenser fan PWM
│   ├── can_protocol.c/.h          # CAN #3 message handling
│   ├── autonomous_mode.c/.h       # CAN loss fallback
│   ├── defrost.c/.h               # Defrost cycle
│   └── fault_handler.c/.h
├── HAL/
│   └── ...                        # STM32F1 or RP2350 HAL
├── Tests/
│   └── ...
└── README.md
```

---

## 6. Shared Protocol Headers

To maintain protocol consistency across repositories, the following headers must be kept in sync:

| Header | Used By | Contents |
|--------|---------|----------|
| `dcfc_safety_can.h` | Safety supervisor + SafetySupervisorBSP | CAN #2 message IDs, signals, fault codes |
| `pdu_can_protocol.h` | PDU-Micro PFC/DAB firmware + PowerModuleDriver | CAN #1 inter-module + CM5 message IDs, signals |
| `hvac_can_protocol.h` | HVAC firmware + HvacDriver | CAN #3 message IDs, signals, fault codes |

**Strategy:** Maintain canonical copies in the firmware repos. Copy (or git submodule) into `dcfc-everest-modules` for the C++ EVerest side.

---

## 7. Risk Register (DCFC-Specific)

| # | Risk | Impact | Likelihood | Mitigation |
|---|------|--------|------------|------------|
| R1 | CAN-FD adapter on CM5 incompatible or unreliable | High | Medium | Verify Waveshare CAN-FD support early (WS-2.2); fallback to dedicated CAN-FD-to-Ethernet gateway (e.g., PEAK PCAN) |
| R2 | PDU-Micro module firmware delayed | Critical | Medium | Start EVerest development with SIL simulation (dcfc_150kw_sim.yaml); decouple CM5 work from hardware |
| R3 | ISO 15118 PLC modem compatibility issues with EVs | High | Medium | Test with multiple EV models early; maintain DIN 70121 fallback |
| R4 | Safety supervisor SIL 2 assessment fails | High | Low | Engage safety assessor early (month 3); design for SIL 2 from start |
| R5 | Thermal management insufficient at 150 kW sustained | Medium | Low | PDU-Micro thermal budget shows adequate margin; validate with full-power test |
| R6 | CM5 compute insufficient for EVerest + ISO 15118 + OCPP | Low | Low | CM5 has 4 GB RAM and quad-core; EVerest is lightweight; PLC modem offloads PHY |
| R7 | OCPP 2.0.1 backend interoperability issues | Medium | Medium | Test with multiple CSMS vendors; use EVerest's certified libocpp |
| R8 | Module-to-module current sharing instability at 5 modules | Medium | Low | PDU-Micro droop-based sharing is well-studied; bench test early |

---

## 8. Team and Skills Required

| Role | Count | Key Skills |
|------|-------|------------|
| Power Electronics Engineer | 1--2 | PDU-Micro hardware, magnetics, thermal (from PDU-Micro team) |
| Embedded Firmware Engineer (safety) | 1 | STM32, bare-metal C, MISRA C, IEC 61508, safety state machines |
| Embedded Firmware Engineer (controls) | 1--2 | dsPIC33CH, power converter control loops (from PDU-Micro team) |
| Linux/EVerest Software Engineer | 1--2 | C++, Linux, EVerest framework, SocketCAN, MQTT |
| Protocol Engineer | 1 | ISO 15118, OCPP 2.0.1, CAN bus, PLC |
| Electrical/Mechanical Engineer | 1 | Cabinet design, busbar, wiring, IP54 enclosure |
| HVAC Engineer | 1 | Refrigeration, compressor control (can be shared/part-time) |
| QA / Certification | 1 | IEC 61851, IEC 62368, EMC, test planning |

**Total:** 8--10 engineers, with 3--4 shared from the existing PDU-Micro team.

---

## 9. Bill of Materials Summary (Cabinet-Level)

Per the existing DCFC hardware component research:

| Item | Unit Cost (est.) | Qty | Subtotal |
|------|----------------:|----:|----------:|
| PDU-Micro 30 kW modules | $1,815 | 5 | $9,075 |
| CM5 4GB + IO Board + PoE HAT | $110 | 1 | $110 |
| Safety Supervisor (STM32 custom board) | $120 | 1 | $120 |
| IO Controller (IRIV IOC RP2350) | $80 | 1 | $80 |
| Waveshare CAN + RS485 adapters | $70 | 2 | $140 |
| PLC modems (ISO 15118) | $200 | 2 | $400 |
| PoE Switch (Teltonika TSW200) | $125 | 1 | $125 |
| PSU (Meanwell SDR-150-24 + DC-DC) | $60 | 1 | $60 |
| UPS batteries (LiFePO4) | $40 | 1 | $40 |
| Display (14" Waveshare TFT + touch) | $170 | 1 | $170 |
| RFID/NFC reader | $50 | 1 | $50 |
| CCS2 connector + liquid-cooled cable (5m) | $1,500 | 1 | $1,500 |
| DC contactors (K2, K3, K4) + pre-charge resistor | $400 | 1 set | $400 |
| AC contactors (2x 200A) + breakers | $300 | 1 set | $300 |
| Backplane busbars + hardware | $200 | 1 | $200 |
| Cabinet enclosure (IP54) | $800 | 1 | $800 |
| HVAC clip-on unit | $1,200 | 1 | $1,200 |
| Coolant pump + radiator + fans | $300 | 1 set | $300 |
| Wiring, connectors, misc. | $500 | 1 | $500 |
| **Total (prototype)** | | | **~$15,570** |

---

## 10. Open Decisions

These items need resolution before or during early development:

| # | Decision | Options | Recommendation | Deadline |
|---|----------|---------|----------------|----------|
| D1 | CM5 to module CAN architecture | Option A (master only) vs Option B (address each) | Option A -- simpler CM5 code, leverages PDU-Micro stacking | Month 1 |
| D2 | CAN-FD adapter for CM5 | Waveshare (verify FD support) vs PEAK PCAN-USB FD vs MCP2518FD SPI hat | Test Waveshare first; order PEAK as backup | Month 1 |
| D3 | Safety supervisor necessity | Full STM32 supervisor vs hardware interlock only | Full supervisor -- required for SIL 2 and contactor sequencing | Month 1 |
| D4 | BSP driver pattern | Pattern B (native CAN) vs Pattern D (MQTT bridge) | Pattern B -- lower latency, simpler stack | Month 2 |
| D5 | Dual connector support | Single CCS2 vs dual CCS2 power-shared | Single first; design for dual extensibility | Month 2 |
| D6 | 5G modem selection | Quectel RM520N vs Sierra Wireless EM9191 vs Telit FN990 | Evaluate by month 4 | Month 4 |
| D7 | HVAC controller platform | STM32F103 vs RP2350 | RP2350 if CAN peripheral is sufficient; STM32F103 as safe choice | Month 2 |
| D8 | PLC modem | QCA7005 (Qualcomm) vs IS32CG5317 (Lumissil) vs PHYTEC module | QCA7005 (widest EV compatibility); PHYTEC as backup | Month 3 |

---

## 11. Related Documents

- [[PDU-Micro/__init]] -- Power module specifications and project plan
- [[__init]] -- DCFC system overview and specifications
- [[docs/System/01 - System Architecture]] -- 4-level hierarchy
- [[docs/Software/03 - Safety Supervisor Controller]] -- Safety FSM specification
- [[docs/Software/04 - Power Module CAN Bus Interface]] -- CAN #1 protocol (to be updated)
- [[docs/Software/EVerest/02 - EVerest Power Module Driver]] -- EVerest module design
- [[docs/HVAC/04 - HVAC CANBus Interface Specification]] -- CAN #3 protocol
- [[research/02 - CM5 based Main Controller]] -- Controller selection rationale
