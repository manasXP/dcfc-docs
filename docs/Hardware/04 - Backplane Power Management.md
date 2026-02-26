# Backplane Power Management

Tags: #dcfc #hardware #power-distribution #backplane #pdu

Related: [[01 - Hardware Components]] | [[02 - Electric Wiring Diagram]] | [[03 - Cabinet Layout]]

## 1. Overview

The backplane power management system is the internal power distribution backbone of the DCFC panel. It receives conditioned AC power from the main input protection stage and distributes it through Power Distribution Units (PDUs) to every subsystem within the cabinet. The backplane must handle both high-power feeds (to power conversion modules) and low-power auxiliary feeds (to control electronics, cooling, HMI, and communication systems), while providing isolation, protection, and sequenced startup for each load.

## 2. Backplane Architecture

```
                         GRID AC INPUT (3-Phase 400/480V)
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │     MAIN INPUT PROTECTION     │
                    │  Main Disconnect ─► SPD ─►    │
                    │  AC Breaker ─► EMI Filter     │
                    │  Energy Meter (MID)           │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
            ┌───────────────────────────────────────────────┐
            │              BACKPLANE POWER BUS              │
            │     (3-Phase + Neutral + PE Busbar System)    │
            │                                               │
            │   L1 ═══════════════════════════════════════  │
            │   L2 ═══════════════════════════════════════  │
            │   L3 ═══════════════════════════════════════  │
            │   N  ═══════════════════════════════════════  │
            │   PE ═══════════════════════════════════════  │
            └───┬──────────┬──────────┬──────────┬─────────┘
                │          │          │          │
                ▼          ▼          ▼          ▼
          ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
          │  PDU 1   │ │  PDU 2   │ │  PDU 3   │ │  PDU 4   │
          │  POWER   │ │  AUX     │ │  COOLING │ │  COMMS   │
          │  MODULES │ │  CONTROL │ │  SYSTEM  │ │  & HMI   │
          └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

## 3. Backplane Bus Components

### 3.1 Busbar System

The backplane uses a copper busbar system mounted at the rear of the cabinet to distribute power from the main input protection stage to each PDU tap point.

| Parameter | Specification | Notes |
|-----------|---------------|-------|
| Material | Electrolytic copper (C110), tin-plated | Corrosion resistance |
| Busbar Cross-Section | 30×5 mm (L1, L2, L3) | Rated for 400A continuous |
| Neutral Bar | 20×5 mm | Sized for unbalanced loads |
| PE Bar | 30×5 mm | Bonded to cabinet chassis |
| Mounting | Insulated standoffs (10 kV rated) | Maintains creepage distance |
| Connections | Bolted joints with Belleville washers | Maintains torque under thermal cycling |
| Temperature Rise | <30°C above ambient at rated load | Per IEC 61439 |

### 3.2 Busbar Layout

```
    REAR CABINET WALL (Vertical Mounting)
    ┌─────────────────────────────────────────────────────────┐
    │                                                         │
    │  PE ══╤══════════╤══════════╤══════════╤═══════════     │
    │       │          │          │          │                │
    │  N  ══╤══════════╤══════════╤══════════╤═══════════     │
    │       │          │          │          │                │
    │  L3 ══╤══════════╤══════════╤══════════╤═══════════     │
    │       │          │          │          │                │
    │  L2 ══╤══════════╤══════════╤══════════╤═══════════     │
    │       │          │          │          │                │
    │  L1 ══╤══════════╤══════════╤══════════╤═══════════     │
    │       │          │          │          │                │
    │    TAP 1      TAP 2     TAP 3      TAP 4                │
    │   (PDU 1)    (PDU 2)   (PDU 3)    (PDU 4)               │
    │                                                         │
    │  Insulated standoff mounting points: ● (every 200mm)    │
    │                                                         │
    └─────────────────────────────────────────────────────────┘
```

### 3.3 Busbar Protection

- **Insulated covers**: Polycarbonate snap-on covers over all live busbars (finger-safe IP2X)
- **Phase barriers**: Insulating dividers between L1, L2, L3 to prevent phase-to-phase flashover
- **Touch-proof tap connectors**: Shrouded bolt connections at each PDU tap point
- **Torque markers**: Visual indicators on all bolted joints for maintenance verification

## 4. Power Distribution Units (PDUs)

Each PDU is a self-contained distribution panel mounted on DIN rail below the busbar tap point. PDUs contain branch protection (MCBs/fuses), contactors, and monitoring for their respective load group.

### 4.1 PDU 1 — Power Modules (High Power)

**Purpose**: Feed 3-phase AC to power conversion modules

```
Backplane Tap 1 (L1, L2, L3, PE)
│
├─── Branch Breaker CB-PM1 (175A, 3-pole, Type C)
│    │
│    └─── AC Contactor K-PM1 (200A, 3-pole)
│         Coil: 24V DC (from safety relay chain)
│         Aux contacts: 1 NO + 1 NC (status feedback)
│         │
│         └─── Power Module #1 AC Input
│
├─── Branch Breaker CB-PM2 (175A, 3-pole, Type C)
│    │
│    └─── AC Contactor K-PM2 (200A, 3-pole)
│         Coil: 24V DC (from safety relay chain)
│         │
│         └─── Power Module #2 AC Input (if installed)
│
└─── Current Monitoring
     3x CT (200A/5A) per branch
     Output: To Main ECU analog inputs
```

| Parameter | Value |
|-----------|-------|
| Max Load per Branch | 175A per phase |
| Total PDU Capacity | 350A (2 branches) |
| Protection Coordination | CB-PM trips before main AC breaker |
| Contactor Control | Via safety relay + Main ECU DO |

### 4.2 PDU 2 — Auxiliary Control (Low Voltage Generation)

**Purpose**: Generate low-voltage DC rails for all control subsystems

```
Backplane Tap 2 (L1, N, PE)  ── Single phase tap
│
├─── Branch Breaker CB-AUX1 (16A, 1-pole, Type C)
│    │
│    └─── SMPS #1 (Primary 24V Supply)
│         Input: 230V AC (L1-N)
│         Output: 24V DC, 10A (240W)
│         │
│         ├─── 24V DC Main Bus ──► To PDU 3, PDU 4
│         ├─── DC-DC Converter: 24V → 12V, 5A (60W)
│         │    └─── 12V Rail ──► Display, RFID, sensors
│         └─── DC-DC Converter: 24V → 5V, 3A (15W)
│              └─── 5V Rail ──► USB devices, logic ICs
│
├─── Branch Breaker CB-AUX2 (16A, 1-pole, Type C)
│    │
│    └─── SMPS #2 (Redundant 24V Supply)
│         Output: 24V DC, 10A (240W)
│         │
│         └─── 24V DC Backup Bus (diode-OR'd with SMPS #1)
│              Purpose: Redundant supply for safety-critical loads
│
└─── UPS / Battery Backup Module
     Input: 24V DC bus
     Battery: 24V LiFePO4, 10Ah
     Output: 24V DC (uninterruptible)
     Runtime: ~10 minutes at full aux load
     Loads: Main ECU, safety relay, communication module
```

| Parameter | Value |
|-----------|-------|
| Primary 24V Supply | 240W (10A) |
| Redundant 24V Supply | 240W (10A), diode-OR'd |
| 12V Rail | 60W (5A) |
| 5V Rail | 15W (3A) |
| UPS Backup | 10 min runtime |

### 4.3 PDU 3 — Cooling System

**Purpose**: Power thermal management components (pump, fans, HVAC interface)

```
24V DC Bus (from PDU 2)
│
├─── Fuse F-PUMP (10A, automotive blade)
│    │
│    └─── Coolant Pump Motor Driver
│         Type: MOSFET H-bridge (PWM control)
│         Load: Brushless DC pump, 24V, 5A max
│         Control: PWM from Main ECU (25 kHz)
│         Feedback: Tachometer pulse to ECU
│
├─── Fuse F-FAN1 (6A, automotive blade)
│    │
│    └─── Radiator Fan #1 Driver
│         Load: Axial fan, 24V, 2A
│         Control: PWM from Main ECU
│
├─── Fuse F-FAN2 (6A, automotive blade)
│    │
│    └─── Radiator Fan #2 Driver
│         Load: Axial fan, 24V, 2A
│         Control: PWM from Main ECU
│
└─── HVAC Clip-On Interface
     │
     ├─── Power: 24V DC feed (fused, 6A)
     ├─── CAN Bus: CANH + CANL (to HVAC controller)
     └─── Quick-disconnect connector at clip-on interface
```

| Parameter | Value |
|-----------|-------|
| Total PDU 3 Load | ~300W max (24V, 12.5A) |
| Pump Current | 5A max |
| Fan Current | 2A each (4A total) |
| HVAC Interface | 24V + CAN, 6A fused |

### 4.4 PDU 4 — Communications & HMI

**Purpose**: Power user interface, networking, and vehicle communication hardware

```
24V DC Bus (from PDU 2) + 12V DC Rail
│
├─── 24V Branch
│    │
│    ├─── Fuse F-ECU (4A)
│    │    └─── Main Controller (SBC/PLC)
│    │         Load: 24V, 2A typical
│    │
│    ├─── Fuse F-LED (2A)
│    │    └─── Status LED drivers
│    │         Load: 24V, <0.5A
│    │
│    └─── Fuse F-ESTOP (2A)
│         └─── E-Stop LED illumination / buzzer
│
├─── 12V Branch
│    │
│    ├─── Fuse F-DISP (3A)
│    │    └─── Touchscreen Display (12V, 2A)
│    │
│    ├─── Fuse F-RFID (1A)
│    │    └─── RFID/NFC Reader (12V, 200mA)
│    │
│    └─── Fuse F-ISO (2A)
│         └─── ISO 15118 PLC Modem (12V, 1A)
│
└─── 5V Branch
     │
     ├─── Fuse F-4G (2A)
     │    └─── Cellular Modem (5V, 2A via buck converter)
     │
     └─── Fuse F-ETH (1A)
          └─── Ethernet Switch / Gateway (5V, 0.5A)
```

| Parameter | Value |
|-----------|-------|
| Total PDU 4 Load | ~120W max |
| 24V loads | ~60W |
| 12V loads | ~40W |
| 5V loads | ~15W |

## 5. Power Sequencing

The backplane must energize subsystems in a controlled sequence to prevent inrush damage, ensure safety circuits are active before power systems engage, and allow the Main ECU to verify each stage before proceeding.

### 5.1 Startup Sequence

```
STEP 1: AC Power Applied (Main Disconnect Closed)
    │
    ├─── Backplane busbars energized
    ├─── Energy meter active (passive, always-on)
    │
    ▼
STEP 2: PDU 2 Energizes (Auxiliary Power)
    │
    ├─── SMPS #1 and SMPS #2 start → 24V DC bus live
    ├─── 12V and 5V DC-DC converters start
    ├─── UPS battery begins charging (if depleted)
    ├─── Main ECU boots (5-15 sec boot time)
    │
    ▼
STEP 3: Safety System Self-Test
    │
    ├─── Main ECU verifies safety relay chain integrity
    ├─── E-Stop circuit verified (NC contacts closed)
    ├─── Door interlock verified
    ├─── IMD (Insulation Monitoring Device) self-test
    ├─── Ground fault detector verified
    │
    ▼
STEP 4: PDU 4 Subsystems Initialize
    │
    ├─── HMI display boots → shows "Initializing"
    ├─── RFID reader initializes
    ├─── Network module connects (OCPP backend)
    ├─── ISO 15118 modem enters standby
    │
    ▼
STEP 5: PDU 3 Cooling System Primed
    │
    ├─── Coolant pump runs at low speed (self-test)
    ├─── Fan tachometer verified
    ├─── HVAC clip-on controller handshake via CAN
    ├─── Coolant flow and temperature sensors validated
    │
    ▼
STEP 6: PDU 1 Power Modules Enabled (On Demand)
    │
    ├─── Main ECU commands K-PM1 / K-PM2 contactors closed
    ├─── Pre-charge sequence for DC link capacitors
    ├─── Power modules enter standby
    ├─── System state: READY (awaiting vehicle plug-in)
    │
    ▼
SYSTEM OPERATIONAL — Ready to accept charging session
```

### 5.2 Shutdown Sequence

| Step | Action | Condition |
|------|--------|-----------|
| 1 | Ramp down charging current to zero | Charging session ends or E-Stop |
| 2 | Open DC output contactors K2, K3 | Current < 5A verified |
| 3 | Open power module contactors K-PM1/2 | DC link discharged |
| 4 | Cooling system runs at idle (post-cool) | 2-5 min thermal soak |
| 5 | PDU 3 cooling system off | Module temps < 45°C |
| 6 | PDU 1 fully de-energized | Contactors open confirmed |
| 7 | PDU 4 enters low-power standby | Network heartbeat maintained |
| 8 | System enters idle mode | Awaiting next session |

### 5.3 Emergency Shutdown (E-Stop / Fault)

```
FAULT or E-STOP DETECTED
    │
    ├─── Safety relay K10 de-energizes (< 50 ms)
    │    ├─── All power module contactors open immediately
    │    ├─── DC output contactors open
    │    └─── AC main contactor K1 opens
    │
    ├─── PDU 1 fully isolated from backplane
    │
    ├─── PDU 2/3/4 remain powered (auxiliary systems stay live)
    │    ├─── Main ECU logs fault condition
    │    ├─── HMI displays fault message
    │    ├─── Cooling continues for thermal protection
    │    └─── OCPP reports fault to backend
    │
    └─── Recovery requires manual E-Stop reset + ECU clear
```

## 6. Monitoring and Diagnostics

### 6.1 Backplane Monitoring Points

| Measurement Point | Sensor Type | Range | Connected To |
|--------------------|-------------|-------|--------------|
| Busbar L1 voltage | Voltage divider | 0-690V AC | Main ECU AI |
| Busbar L2 voltage | Voltage divider | 0-690V AC | Main ECU AI |
| Busbar L3 voltage | Voltage divider | 0-690V AC | Main ECU AI |
| PDU 1 current (per phase) | CT (200A/5A) | 0-250A | Main ECU AI |
| 24V DC bus voltage | Direct sense | 0-30V DC | Main ECU AI |
| 24V DC bus current | Shunt (100mV/10A) | 0-15A | Main ECU AI |
| UPS battery voltage | Direct sense | 20-29V DC | Main ECU AI |
| UPS charge state | SOC estimate | 0-100% | Main ECU |
| Busbar temperature | NTC thermistor | -20 to +120°C | Main ECU AI |

### 6.2 Fault Conditions

| Fault | Detection Method | Response |
|-------|------------------|----------|
| Phase loss (L1/L2/L3) | Voltage monitoring < 340V | Derate or shutdown power modules |
| 24V bus undervoltage | Voltage < 21V | Switch to UPS, alarm |
| 24V bus overvoltage | Voltage > 28V | Disconnect SMPS, crowbar circuit |
| PDU overcurrent | CT reading > 110% rated | Trip branch breaker |
| Busbar over-temperature | NTC > 85°C | Derate power, alarm |
| UPS battery low | SOC < 20% | Prevent new sessions, alarm |
| Ground fault on DC bus | IMD reading < threshold | Emergency shutdown |
| SMPS failure | Redundant supply takeover | Alarm, continue on backup |

### 6.3 Diagnostic Reporting

All backplane monitoring data is:
- Logged locally on Main ECU (rotating 30-day log)
- Reported via OCPP `MeterValues` and `DiagnosticsStatusNotification`
- Available on HMI diagnostic screen (service access only)
- Exportable via USB or remote download for field service analysis

## 7. Physical Layout Within Cabinet

```
┌───────────────────────────────────────────────────────────────────┐
│                          REAR CABINET WALL                        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    BUSBAR ASSEMBLY                          │  │  ▲
│  │   L1 ═══╤═══════╤═══════╤═══════╤═══                        │  │  │
│  │   L2 ═══╤═══════╤═══════╤═══════╤═══                        │  │  80mm
│  │   L3 ═══╤═══════╤═══════╤═══════╤═══                        │  │  │
│  │   N  ═══╤═══════╤═══════╤═══════╤═══                        │  │  │
│  │   PE ═══╤═══════╤═══════╤═══════╤═══                        │  │  ▼
│  └─────────┼───────┼───────┼───────┼───────────────────────────┘  │
│            │       │       │       │                              │
│  ┌─────────▼──┐ ┌──▼──────┐ ┌─────▼────┐ ┌──────▼────┐            │
│  │   PDU 1    │ │  PDU 2  │ │  PDU 3   │ │   PDU 4   │            │  ▲
│  │  POWER     │ │   AUX   │ │ COOLING  │ │  COMMS    │            │  │
│  │  MODULES   │ │ CONTROL │ │  SYSTEM  │ │  & HMI    │            │  │
│  │            │ │         │ │          │ │           │            │  │
│  │ CB-PM1     │ │ SMPS #1 │ │ F-PUMP   │ │ F-ECU     │            │  │
│  │ K-PM1      │ │ SMPS #2 │ │ F-FAN1   │ │ F-DISP    │            │  300mm
│  │ CB-PM2     │ │ UPS     │ │ F-FAN2   │ │ F-RFID    │            │  │
│  │ K-PM2      │ │ DC-DC   │ │ F-HVAC   │ │ F-ISO     │            │  │
│  │ CTs        │ │ DC-DC   │ │          │ │ F-4G      │            │  │
│  │            │ │         │ │          │ │ F-ETH     │            │  │
│  └────────────┘ └─────────┘ └──────────┘ └───────────┘            │  ▼
│                                                                   │
│  DIN Rail mounting for all PDUs                                   │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### 7.1 Placement Considerations

- **PDU 1** positioned closest to power module AC input terminals to minimize high-current cable runs
- **PDU 2** centered for shortest 24V distribution paths to PDU 3 and PDU 4
- **PDU 3** positioned near cooling zone (pump, fans) at bottom of cabinet
- **PDU 4** positioned near control electronics zone at top of cabinet
- **Busbar assembly** runs horizontally along rear wall, above the AC input/protection zone
- Minimum **100mm vertical clearance** between busbars and PDU DIN rails for heat dissipation

## 8. Wiring Between Backplane and PDUs

### 8.1 High-Power Connections (Backplane → PDU 1)

| Connection | Wire | Size | Color | Length |
|------------|------|------|-------|--------|
| L1 tap → CB-PM1 | XLPE | 95 mm² | Brown | <500mm |
| L2 tap → CB-PM1 | XLPE | 95 mm² | Black | <500mm |
| L3 tap → CB-PM1 | XLPE | 95 mm² | Gray | <500mm |
| PE tap → PDU 1 bar | PVC | 35 mm² | Green/Yellow | <500mm |
| L1 tap → CB-PM2 | XLPE | 95 mm² | Brown | <500mm |
| L2 tap → CB-PM2 | XLPE | 95 mm² | Black | <500mm |
| L3 tap → CB-PM2 | XLPE | 95 mm² | Gray | <500mm |

### 8.2 Auxiliary Connections (Backplane → PDU 2)

| Connection | Wire | Size | Color | Length |
|------------|------|------|-------|--------|
| L1 tap → CB-AUX1 | PVC | 4 mm² | Brown | <400mm |
| N tap → CB-AUX1 | PVC | 4 mm² | Blue | <400mm |
| L1 tap → CB-AUX2 | PVC | 4 mm² | Brown | <400mm |
| N tap → CB-AUX2 | PVC | 4 mm² | Blue | <400mm |
| PE tap → PDU 2 bar | PVC | 4 mm² | Green/Yellow | <400mm |

### 8.3 24V DC Distribution (PDU 2 → PDU 3, PDU 4)

| Connection | Wire | Size | Color |
|------------|------|------|-------|
| 24V+ bus → PDU 3 | PVC | 4 mm² | Red |
| 0V bus → PDU 3 | PVC | 4 mm² | Black |
| 24V+ bus → PDU 4 | PVC | 2.5 mm² | Red |
| 0V bus → PDU 4 | PVC | 2.5 mm² | Black |
| 12V rail → PDU 4 | PVC | 1.5 mm² | Orange |
| 5V rail → PDU 4 | PVC | 1.5 mm² | White |

## 9. Bill of Materials — Backplane Power Management

### 9.1 Busbar Assembly

| Item | Qty | Specification |
|------|-----|---------------|
| Copper busbar (L1, L2, L3) | 3 | 30×5mm, 700mm length, tin-plated |
| Copper busbar (N) | 1 | 20×5mm, 700mm length, tin-plated |
| Copper busbar (PE) | 1 | 30×5mm, 700mm length, tin-plated |
| Insulated standoffs | 20 | M8, 10kV rated, polyester |
| Phase barrier plates | 4 | Polycarbonate, 700×30mm |
| Busbar covers | 5 | IP2X finger-safe, polycarbonate |
| Tap connectors (bolted) | 16 | M8 brass bolts, Belleville washers |

### 9.2 PDU Components

| Item | Qty | Specification |
|------|-----|---------------|
| 3-pole MCB 175A Type C | 2 | PDU 1 branch breakers |
| 3-pole AC contactor 200A | 2 | PDU 1 power module feeds |
| 1-pole MCB 16A Type C | 2 | PDU 2 SMPS feeds |
| SMPS 24V/10A | 2 | PDU 2 primary + redundant |
| DC-DC 24V→12V/5A | 1 | PDU 2 |
| DC-DC 24V→5V/3A | 1 | PDU 2 |
| UPS module 24V/10Ah | 1 | PDU 2 |
| Diode-OR module | 1 | PDU 2 redundancy |
| Automotive blade fuses (assorted) | 12 | PDU 3 and PDU 4 |
| Fuse holders | 12 | DIN rail mount |
| CTs 200A/5A | 6 | PDU 1 (3 per branch) |
| DIN rail (35mm) | 4m | PDU mounting |
| Terminal blocks (assorted) | 40 | DIN rail mount |

## 10. Design Considerations

### 10.1 Redundancy

- **Dual SMPS** with diode-OR output ensures no single PSU failure takes down the control system
- **UPS battery** provides ride-through for grid disturbances and graceful shutdown on extended outage
- **Independent branch protection** per power module allows one module to fault without affecting the other

### 10.2 Thermal Derating

The backplane busbar system should be derated for ambient temperatures above 40°C:

| Ambient Temp | Busbar Derating Factor |
|--------------|------------------------|
| ≤40°C | 1.00 (100%) |
| 45°C | 0.95 (95%) |
| 50°C | 0.90 (90%) |
| 55°C | 0.82 (82%) |

### 10.3 Selectivity (Protection Coordination)

The branch breakers in each PDU must trip before the main AC breaker to ensure faults are isolated to the affected subsystem without de-energizing the entire panel:

```
Main AC Breaker (350A) ─► PDU 1 Branch (175A) ─► Power Module Fuse
                        ─► PDU 2 Branch (16A)  ─► SMPS internal fuse
```

Time-current curves must be coordinated during design to guarantee selectivity across all fault levels up to the prospective short-circuit current at the panel.

## 11. References

- IEC 61439-1/2: Low-voltage switchgear and controlgear assemblies
- IEC 60947-2: Circuit breakers for overcurrent protection
- [[01 - Hardware Components]] — Component specifications
- [[02 - Electric Wiring Diagram]] — Full electrical schematic
- [[03 - Cabinet Layout]] — Physical placement and thermal zones

---

**Document Version**: 1.0
**Last Updated**: 2026-02-26
**Prepared by**: Electrical Engineering
