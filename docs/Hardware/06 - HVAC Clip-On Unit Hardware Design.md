# HVAC Clip-On Unit Hardware Design

Tags: #dcfc #hardware #hvac #thermal #cooling #mechanical

Related: [[01 - Hardware Components]] | [[03 - Cabinet Layout]] | [[04 - Backplane Power Management]] | [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]] | [[docs/System/01 - System Architecture|01 - System Architecture]]

## 1. Overview

The HVAC clip-on unit is a self-contained, externally mounted cooling system that manages the thermal environment inside the DCFC cabinet. Unlike traditional vented enclosures that draw ambient air through the electronics, this design creates a **closed-loop internal air circuit** — hot air is drawn from the cabinet, passed over a refrigeration evaporator, and returned as cooled air. External ambient air never contacts the electronics, enabling a sealed cabinet rated IP55 or higher.

The clip-on form factor means the HVAC unit is a field-replaceable unit (FRU) that can be swapped without opening the main cabinet or disturbing power electronics. See [[research/03 - 150kW DCFC Comparison#Design Decision Air Cooling with Clip-On HVAC Unit]] for the design rationale behind choosing this approach over liquid cooling.

## 2. System Context

```
                        MAIN CHARGING CABINET
    ┌────────────────────────────────────────────────────────┐
    │                                                        │
    │   ┌───────────────┐   ┌──────────────┐                 │
    │   │ User Interface│   │   Control    │  Zone 1: 35-45°C│
    │   │   Panel       │   │  Electronics │                 │
    │   └───────────────┘   └──────────────┘                 │
    │                                                        │
    │   ┌──────────────────────────────────┐                 │
    │   │         Power Modules            │  Zone 2: 55-70°C│
    │   │  (Primary heat source: 3-15 kW)  │                 │
    │   └──────────────────────────────────┘                 │
    │                                                        │
    │   ┌──────────────────────────────────┐                 │
    │   │    AC Input / Protection         │  Zone 4: 35-50°C│
    │   └──────────────────────────────────┘                 │
    │                                                        │
    │   HOT AIR ═══════════════════════════► OUT PORT ──┐    │
    │                                                   │    │
    │   COLD AIR ◄═══════════════════════════ IN PORT ──┼──┐ │
    │                                                   │  │ │
    └───────────────────────────────────────────────────┼──┼─┘
                                                        │  │
    ┌───────────────────────────────────────────────────┼──┼─┐
    │              HVAC CLIP-ON UNIT                    │  │ │
    │                                                   │  │ │
    │   HOT AIR IN ◄────────────────────────────────────┘  │ │
    │        │                                             │ │
    │   ┌────▼──────────────┐                              │ │
    │   │   Evaporator Coil │  (Absorbs heat from          │ │
    │   │   + Internal Fan  │   cabinet air)               │ │
    │   └────┬──────────────┘                              │ │
    │        │                                             │ │
    │   COLD AIR OUT ──────────────────────────────────────┘ │
    │                                                        │
    │   ┌───────────────────┐   ┌────────────────────┐       │
    │   │  Compressor       │   │  Condenser Coil    │       │
    │   │  (Variable Speed) │──►│  + External Fan    │──► EXHAUST
    │   └───────────────────┘   └────────────────────┘   TO AMBIENT
    │                                                        │
    │   ┌───────────────────┐                                │
    │   │  HVAC Controller  │  MCU + CAN interface           │
    │   │  (STM32 / RP2350) │  to Phytec SBC Main Controller        │
    │   └───────────────────┘                                │
    │                                                        │
    └────────────────────────────────────────────────────────┘
```

## 3. Thermal Load Analysis

### 3.1 Heat Sources Within the Cabinet

| Component | Heat Dissipation (150 kW charger) | Heat Dissipation (350 kW charger) | Notes |
|-----------|-----------------------------------|-----------------------------------|-------|
| Power modules (at 96% eff.) | 6 kW | 14 kW | Dominant heat source |
| DC contactors (K2, K3) | 0.25 kW | 0.5 kW | Contact resistance losses |
| AC contactor (K1) | 0.1 kW | 0.2 kW | |
| DC breaker + fuses | 0.15 kW | 0.3 kW | |
| Control electronics | 0.1 kW | 0.1 kW | ECU, modem, display |
| Auxiliary PSUs (SMPS, DC-DC) | 0.05 kW | 0.1 kW | |
| Wiring and busbar losses | 0.1 kW | 0.3 kW | |
| **Total internal heat load** | **~6.75 kW** | **~15.5 kW** | |

### 3.2 HVAC Sizing

The HVAC unit must reject the total internal heat load plus account for solar load and ambient heat ingress through the cabinet walls.

| Factor | 150 kW System | 350 kW System |
|--------|---------------|---------------|
| Internal heat load | 6.75 kW | 15.5 kW |
| Solar load (direct sun on cabinet) | 0.3–0.5 kW | 0.3–0.5 kW |
| Ambient heat ingress (50°C outside, 35°C target inside) | 0.2–0.5 kW | 0.3–0.7 kW |
| **Total cooling required** | **~7.5 kW** | **~16.5 kW** |
| **HVAC sizing (with 20% margin)** | **~9 kW** | **~20 kW** |

> [!note] Scaling
> The CAN interface specification in [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]] references 1.5–3 kW cooling capacity for the initial 150 kW design target with partial power module loading. The full 150 kW continuous load sizing is ~9 kW. The 350 kW variant documented in [[docs/System/01 - System Architecture|System Architecture]] (15–25 kW thermal) aligns with the full-load analysis here.

## 4. Refrigeration System

### 4.1 Refrigeration Cycle

```
                    HIGH PRESSURE SIDE
                    ────────────────────

    ┌──────────────────────────────────────────────────┐
    │                                                  │
    │   COMPRESSOR ──────────► CONDENSER COIL          │
    │   (High-pressure          (Rejects heat to       │
    │    hot gas)                ambient air via       │
    │                           external fan)          │
    │                                │                 │
    │                           LIQUID LINE            │
    │                           (Warm liquid)          │
    │                                │                 │
    │                      ┌─────────▼──────────┐      │
    │                      │ EXPANSION DEVICE   │      │
    │                      │ (TXV or EEV)       │      │
    │                      │ Pressure drop →    │      │
    │                      │ temp drop          │      │
    │                      └─────────┬──────────┘      │
    │                                │                 │
    │                           LOW PRESSURE           │
    │                           (Cold liquid/gas mix)  │
    │                                │                 │
    │   COMPRESSOR ◄──────────── EVAPORATOR COIL       │
    │   (Low-pressure            (Absorbs heat from    │
    │    cool gas)                cabinet air via      │
    │                             internal fan)        │
    │                                                  │
    └──────────────────────────────────────────────────┘

                    LOW PRESSURE SIDE
                    ────────────────────
```

### 4.2 Refrigerant Selection

| Parameter | R-290 (Propane) | R-134a | R-32 |
|-----------|-----------------|--------|------|
| GWP (Global Warming Potential) | 3 | 1430 | 675 |
| ODP (Ozone Depletion) | 0 | 0 | 0 |
| Charge limit (IEC 60335-2-40) | ~150 g (per enclosure) | No practical limit | No practical limit |
| COP (Coefficient of Performance) | 3.5–4.5 | 3.0–3.5 | 3.5–4.2 |
| Flammability | A3 (highly flammable) | A1 (non-flammable) | A2L (mildly flammable) |
| Suitability for high capacity | Limited by charge size | Good | Good |
| Regulatory trend | Favoured (low GWP) | Being phased out (F-Gas) | Widely adopted |

**Selection**: R-32 for units ≥5 kW cooling capacity (good COP, moderate GWP, widely available components). R-290 for smaller units ≤3 kW where the charge limit is not exceeded (lowest environmental impact).

### 4.3 Compressor

| Parameter | Value (150 kW variant) | Value (350 kW variant) |
|-----------|------------------------|------------------------|
| Type | Rotary or scroll, DC inverter-driven | Scroll, DC inverter-driven |
| Cooling capacity | 9 kW nominal at 35°C ambient | 20 kW nominal at 35°C ambient |
| Electrical input | 2–3 kW | 5–7 kW |
| Power supply | 230V AC (single phase from PDU 2) or 24V DC (brushless) | 230V AC (single phase) |
| Speed range | 1800–7200 RPM (inverter controlled) | 1800–7200 RPM |
| COP at rated conditions | 3.0–3.5 | 3.0–3.5 |
| Refrigerant | R-32 or R-290 | R-32 |
| Oil type | POE (Polyolester) | POE |
| Start method | Soft-start via inverter (no inrush) | Soft-start via inverter |
| Vibration isolation | Rubber grommets, spring mounts | Rubber grommets, spring mounts |
| Sound level | <55 dB(A) at 1 m | <60 dB(A) at 1 m |

### 4.4 Expansion Device

| Parameter | Value |
|-----------|-------|
| Type | Electronic Expansion Valve (EEV) |
| Control | Stepper motor, driven by HVAC controller |
| Superheat target | 5–8°C |
| Response time | <2 s for full stroke |
| Advantage over TXV | Precise control across wide load range, enables variable-speed compressor optimization |

### 4.5 Heat Exchangers

#### Evaporator Coil (Internal — Cabinet Side)

| Parameter | Value |
|-----------|-------|
| Type | Fin-and-tube, copper tube / aluminum fin |
| Face area | 300×200 mm (150 kW) / 400×300 mm (350 kW) |
| Fin pitch | 2.0–2.5 mm |
| Tube diameter | 7 mm or 9.52 mm |
| Rows | 3–4 |
| Coating | Epoxy-coated fins (corrosion protection) |
| Condensate drain | Drip tray with drain hose to exterior |
| Defrost method | Hot-gas bypass or reverse cycle (cold climate) |

#### Condenser Coil (External — Ambient Side)

| Parameter | Value |
|-----------|-------|
| Type | Fin-and-tube or microchannel |
| Face area | 400×300 mm (150 kW) / 500×400 mm (350 kW) |
| Fin pitch | 1.5–2.0 mm |
| Design ambient | Up to 50°C |
| Coating | Hydrophilic coating for condensate management |
| Cleaning access | Removable guard for pressure washing |

## 5. Airflow System

### 5.1 Internal Circulation Fan (Evaporator Fan)

This fan moves cabinet air across the evaporator coil and distributes cooled air back into the cabinet through the cold air plenum.

| Parameter | Value |
|-----------|-------|
| Type | Centrifugal blower (forward-curved or backward-curved) |
| Motor | Brushless DC, external rotor |
| Voltage | 24V DC (from PDU 3) or 230V AC |
| Airflow | 500–1500 CFM (150 kW) / 1500–3500 CFM (350 kW) |
| Static pressure | 100–300 Pa (to overcome duct and coil pressure drop) |
| Speed control | PWM (0–100%), commanded by HVAC controller |
| Power consumption | 50–150W (150 kW) / 150–400W (350 kW) |
| Feedback | Tachometer pulse output |
| Noise | <50 dB(A) at 1 m |
| Bearing type | Ball bearing (>60,000 hr L10 life) |

### 5.2 External Condenser Fan

This fan forces ambient air across the condenser coil to reject heat outdoors.

| Parameter | Value |
|-----------|-------|
| Type | Axial fan |
| Motor | Brushless DC, external rotor |
| Voltage | 24V DC or 230V AC |
| Airflow | 800–2000 CFM |
| Speed control | PWM, modulated based on condenser pressure/temperature |
| Power consumption | 50–200W |
| Feedback | Tachometer pulse output |
| Weather protection | Rain guard / drip shield over intake |
| Bearing type | Ball bearing (>60,000 hr) |

### 5.3 Airflow Path Detail

```
SIDE VIEW (Cut-Away)

        HVAC UNIT                           MAIN CABINET
    ┌─────────────────┐                ┌──────────────────────┐
    │                 │  COLD AIR      │                      │
    │   ┌──────────┐  │ ═══════════►   │  ┌────────────────┐  │
    │   │EVAPORATOR│  │  (5–15°C       │  │  Cold Air      │  │
    │   │  COIL    │  │   below        │  │  Plenum        │  │
    │   │          │  │   cabinet      │  │  (distribution │  │
    │   └──────────┘  │   setpoint)    │  │   baffles)     │  │
    │        ▲        │                │  └───┬───┬───┬────┘  │
    │        │        │                │      │   │   │       │
    │   ┌────┴─────┐  │                │      ▼   ▼   ▼       │
    │   │ INTERNAL │  │                │   ┌─────────────┐    │
    │   │ BLOWER   │  │                │   │   Power     │    │
    │   │ FAN      │  │                │   │   Modules   │    │
    │   └────┬─────┘  │                │   │ (heat source│    │
    │        │        │                │   │  55-70°C)   │    │
    │        │        │  HOT AIR       │   └─────────────┘    │
    │        │        │ ◄═══════════   │          │           │
    │        │        │  (45-55°C)     │   ┌──────┴──────┐    │
    │        │        │                │   │ Hot Air     │    │
    │   ┌────┴─────┐  │                │   │ Return      │    │
    │   │CONDENSATE│  │                │   │ Duct        │    │
    │   │ TRAY     │  │                │   └─────────────┘    │
    │   └──────────┘  │                │                      │
    │                 │                └──────────────────────┘
    │   ┌──────────┐  │
    │   │COMPRESSOR│  │
    │   └──────────┘  │
    │                 │
    │   ┌──────────┐  │
    │   │CONDENSER │  │  ═══► HOT AIR EXHAUST
    │   │  COIL    │  │       TO AMBIENT
    │   │  + FAN   │  │
    │   └──────────┘  │
    │                 │
    └─────────────────┘
```

### 5.4 Air Distribution Within Cabinet

The cold air plenum inside the cabinet uses baffles to direct airflow preferentially toward the highest heat-density zone (power modules). The plenum is integrated into the cabinet structure, not part of the HVAC unit itself.

```
TOP VIEW — Air Distribution in Cabinet

    ┌──────────────────────────────────────────────────────┐
    │                        FRONT                         │
    │                                                      │
    │   ┌─────────────────────────────────────────────┐    │
    │   │              COLD AIR PLENUM                │    │
    │   │   ┌───────┐  ┌───────┐  ┌───────┐           │    │
    │   │   │ Slot 1│  │ Slot 2│  │ Slot 3│  (Angled  │    │
    │   │   │  ▼    │  │  ▼    │  │  ▼    │   louvers │    │
    │   │   │ 40%   │  │ 40%   │  │ 20%   │   direct  │    │
    │   │   │ flow  │  │ flow  │  │ flow  │   airflow)│    │
    │   │   └───────┘  └───────┘  └───────┘           │    │
    │   └─────────────────────────────────────────────┘    │
    │            │            │           │                │
    │            ▼            ▼           ▼                │
    │   ┌─────────────────────────────────────────────┐    │
    │   │                                             │    │
    │   │   Power Module 1    Power Module 2          │    │
    │   │   (Primary airflow  (Primary airflow        │    │
    │   │    target)           target)                │    │
    │   │                                             │    │
    │   └─────────────────────────────────────────────┘    │
    │                                                      │
    │   Hot air rises → collected at top → to HVAC return  │
    │                                                      │
    │                        REAR                          │
    └──────────────────────────────────────────────────────┘

Air Split:
  - 40% to Power Module 1 zone
  - 40% to Power Module 2 zone
  - 20% to control electronics and AC input zones
```

## 6. HVAC Controller

### 6.1 MCU and Electronics

The HVAC unit contains its own embedded controller that runs the refrigeration cycle autonomously based on commands from the Phytec SBC main controller.

| Parameter | Value |
|-----------|-------|
| MCU | STM32F103 or RP2350 |
| Architecture | Bare-metal or FreeRTOS |
| CAN interface | CAN 2.0B, 250 kbps (to Phytec SBC via CAN #3) |
| ADC channels | 6 (4× temperature, 1× pressure, 1× current) |
| PWM outputs | 3 (compressor inverter, internal fan, external fan) |
| Digital outputs | 3 (EEV stepper, heater relay, fault LED) |
| Digital inputs | 2 (high-pressure switch, low-pressure switch) |
| Power supply | 24V DC input → internal 3.3V/5V regulator |
| Firmware update | CAN bootloader or UART/SWD debug port |
| Watchdog | Internal IWDG; autonomous safe mode on Phytec SBC timeout |

### 6.2 Sensor Inputs

| Sensor | Type | Range | Location | Connected To |
|--------|------|-------|----------|--------------|
| T1: Cabinet air temperature | NTC 10kΩ or PT1000 | -40 to +125°C | Cabinet hot air return duct | HVAC controller AI0 |
| T2: Condenser coil temperature | NTC 10kΩ | -40 to +125°C | Condenser outlet pipe | HVAC controller AI1 |
| T3: Evaporator coil temperature | NTC 10kΩ | -40 to +125°C | Evaporator outlet pipe | HVAC controller AI2 |
| T4: Ambient temperature | NTC 10kΩ | -40 to +85°C | External, shaded from sun | HVAC controller AI3 |
| P1: High-side pressure | Pressure transducer | 0–40 bar | Compressor discharge | HVAC controller AI4 |
| CS1: Compressor current | Hall effect / shunt | 0–15A | Compressor supply line | HVAC controller AI5 |
| HP: High-pressure safety switch | Normally closed | Opens at 28 bar | Compressor discharge line | HVAC controller DI0 |
| LP: Low-pressure safety switch | Normally closed | Opens at 1.5 bar | Compressor suction line | HVAC controller DI1 |

### 6.3 Control Outputs

| Output | Type | Driven Component | Control Method |
|--------|------|-------------------|----------------|
| Compressor speed | PWM / inverter | Rotary/scroll compressor | PID on cabinet temp setpoint |
| Internal fan speed | PWM | Evaporator blower | Proportional to compressor speed + cabinet temp |
| External fan speed | PWM | Condenser axial fan | PID on condenser pressure/temp |
| EEV position | Stepper motor | Electronic expansion valve | PID on superheat target |
| Heater relay | On/off | PTC heater element | On below low-temp threshold |
| Fault LED | On/off | Panel-mount LED on HVAC unit | On during any active fault |

### 6.4 Control Algorithm

```
MAIN CONTROL LOOP (runs every 1 second)
│
├─── READ all sensor inputs (T1-T4, P1, CS1, HP, LP)
│
├─── CHECK safety conditions
│    ├── HP switch open? → Disable compressor, fault 0x03
│    ├── LP switch open? → Disable compressor, fault 0x04
│    ├── CS1 > 110% rated? → Disable compressor, fault 0x01
│    └── T2 > 65°C? → Disable compressor, reduce condenser fan
│
├─── DETERMINE operating mode
│    ├── Mode from Phytec SBC command (CAN 0x200)
│    ├── If Phytec SBC heartbeat lost > 6 s → AUTONOMOUS MODE
│    └── If critical fault → SHUTDOWN MODE
│
├─── EXECUTE mode logic
│    │
│    ├── OFF: All outputs disabled
│    │
│    ├── STANDBY: Fans at minimum (10%), compressor off
│    │   Monitor T1, report status, ready to ramp up
│    │
│    ├── COOLING:
│    │   ┌─── PID Controller: Compressor Speed
│    │   │    Input: T1 (cabinet temp)
│    │   │    Setpoint: From Phytec SBC command (default 35°C)
│    │   │    Output: Compressor RPM (1800-7200)
│    │   │    Kp=10, Ki=0.5, Kd=2 (tunable)
│    │   │
│    │   ├─── PID Controller: EEV Position
│    │   │    Input: T3 - suction temp (superheat)
│    │   │    Setpoint: 6°C superheat
│    │   │    Output: EEV steps (0-500)
│    │   │
│    │   ├─── Internal Fan: Proportional to compressor speed
│    │   │    Min 30% when compressor running
│    │   │
│    │   └─── External Fan: PID on condenser temp
│    │        Higher ambient → higher fan speed
│    │
│    ├── HEATING: (cold climate, T1 < low threshold)
│    │   Heater relay ON, internal fan at 50%
│    │   Compressor OFF
│    │   PID on T1 to cycle heater at setpoint
│    │
│    ├── DEFROST: (T3 < 0°C for >10 min, ice buildup)
│    │   Reverse cycle or hot-gas bypass to evaporator
│    │   Duration: 5-10 min, then return to COOLING
│    │
│    └── VENTILATION: (mild conditions, T1 near setpoint)
│        Fans only, compressor off
│        Energy-saving mode
│
├─── SEND CAN status messages (0x100, 0x101, per schedule)
│
└─── FEED watchdog timer
```

### 6.5 Autonomous Safe Mode

If the HVAC controller loses CAN communication with the Phytec SBC for more than 6 seconds (3 missed heartbeat cycles), it enters autonomous mode:

| Behaviour | Description |
|-----------|-------------|
| Maintain last setpoint | Continue cooling/heating at the last commanded setpoint |
| Compressor at 70% | Cap compressor speed to avoid unnecessary wear |
| Internal fan at 60% | Moderate airflow |
| Fault LED on | Steady red to indicate communication loss |
| Status messages continue | HVAC keeps transmitting on CAN in case Phytec SBC recovers |
| Timeout (30 min) | If no recovery, HVAC transitions to max cooling as precaution |

## 7. Electrical Design

### 7.1 Power Supply

The HVAC unit receives power from the main cabinet through the clip-on interface connector. Two power configurations are supported depending on compressor type:

#### Configuration A: 24V DC HVAC (≤3 kW cooling capacity)

```
From PDU 3 (24V DC Bus)
│
├─── 24V DC (fused at 6A in PDU 3)
│    │
│    ├─── HVAC Controller (24V → 3.3V internal regulator)
│    ├─── Internal blower fan (24V DC, PWM)
│    ├─── External condenser fan (24V DC, PWM)
│    └─── Compressor inverter (24V DC → 3-phase variable frequency)
│         Input: 24V DC, 8A max
│         Output: 3-phase to DC brushless compressor
│
└─── Total HVAC power draw: ≤200W (fans) + 200W (compressor) = ~400W max
```

#### Configuration B: 230V AC HVAC (5–20 kW cooling capacity)

```
From Backplane Tap (L1-L2, via dedicated HVAC breaker)
│
├─── 400V AC L-L (MCB 10A for 9 kW unit / 16A for 20 kW unit)
│    │
│    ├─── Compressor Inverter Drive
│    │    Input: 400V AC, single-phase (L-L)
│    │    Output: 3-phase variable frequency to scroll compressor
│    │    Max input current: 7A (9 kW) / 15A (20 kW)
│    │
│    ├─── PTC Heater Element (optional, cold climate)
│    │    Rating: 500W–1 kW, 400V AC
│    │    Switched by HVAC controller relay
│    │
│    └─── Internal buck converter: 400V AC → 24V DC, 2A
│         Powers: controller MCU, fan motors, EEV stepper, sensors
│
├─── 24V DC (from PDU 3, via clip-on connector)
│    │
│    ├─── Backup power for controller (if AC is lost)
│    ├─── Fan motors (can run on either AC-derived 24V or PDU 3 24V)
│    └─── CAN interface power
│
└─── Total HVAC power draw:
     9 kW unit:  ~3 kW electrical (COP ~3.0)
     20 kW unit: ~7 kW electrical (COP ~2.8)
```

### 7.2 Electrical Protection Within HVAC Unit

| Device | Rating | Purpose |
|--------|--------|---------|
| Input fuse (AC) | 16A or 32A gG | Compressor overcurrent protection |
| Input fuse (24V DC) | 10A | Controller and fan protection |
| Compressor thermal overload | Built into compressor windings | Winding over-temperature |
| High-pressure switch (HP) | NC, opens at 28 bar | Compressor discharge overpressure |
| Low-pressure switch (LP) | NC, opens at 1.5 bar | Suction pressure too low (leak/blockage) |
| MOV surge suppressor | 230V AC line | Transient protection |
| EMI filter | On AC input | CE EMC compliance |

### 7.3 Wiring Within HVAC Unit

| Connection | Wire | Size | Notes |
|------------|------|------|-------|
| AC input → inverter | H07RN-F | 2.5 mm² (16A) / 6 mm² (32A) | Flexible rubber |
| Inverter → compressor | Shielded 3-core | 1.5 mm² | Short run (<1 m) |
| 24V bus → fans | PVC | 1.5 mm² | With inline fuse |
| Controller → EEV | Stepper cable (4-core) | 0.5 mm² | Shielded |
| Sensor wires | Shielded pair | 0.5 mm² | Star grounded at controller |
| CAN bus (internal) | Twisted pair shielded | 0.75 mm² | To M12 connector |

## 8. Mechanical Design

### 8.1 HVAC Unit Enclosure

| Parameter | 150 kW Variant | 350 kW Variant |
|-----------|----------------|----------------|
| Dimensions (H×W×D) | 600×400×300 mm | 800×500×400 mm |
| Weight | 25–35 kg | 45–65 kg |
| Material | Powder-coated galvanized steel | Powder-coated galvanized steel |
| IP Rating | IP55 (unit itself) | IP55 |
| Color | RAL 7035 (light gray) to match cabinet | RAL 7035 |
| Mounting position | Side, rear, or top of cabinet | Side or rear |

### 8.2 Clip-On Interface

The clip-on interface is the mechanical and electrical boundary between the HVAC unit and the main cabinet. It is designed for tool-free HVAC attachment (aside from 4 securing bolts) and quick-disconnect of all services.

```
CLIP-ON INTERFACE (Face View — HVAC Side)

    ┌────────────────────────────────────────────────────┐
    │                                                    │
    │   ┌──────────────────┐   ┌──────────────────┐      │
    │   │                  │   │                  │      │
    │   │   HOT AIR        │   │   COLD AIR       │      │
    │   │   INTAKE PORT    │   │   SUPPLY PORT    │      │
    │   │   (from cabinet) │   │   (to cabinet)   │      │
    │   │                  │   │                  │      │
    │   │   200×150 mm     │   │   200×150 mm     │      │
    │   │                  │   │                  │      │
    │   └──────────────────┘   └──────────────────┘      │
    │                                                    │
    │   ┌───────┐   ┌───────┐   ┌────────┐               │
    │   │ POWER │   │  CAN  │   │ DRAIN  │               │
    │   │ CONN  │   │ M12   │   │ HOSE   │               │
    │   │(AC+DC)│   │4-pin  │   │ (cond.)│               │
    │   └───────┘   └───────┘   └────────┘               │
    │                                                    │
    │   ●       ●               ●       ●                │
    │  (M8 mounting bolt holes, 4× with alignment pins)  │
    │                                                    │
    └────────────────────────────────────────────────────┘
```

#### Interface Components

| Component | Specification | Purpose |
|-----------|---------------|---------|
| Air ports (×2) | 200×150 mm openings with foam gasket seal | Hot air intake, cold air supply |
| Foam gaskets | Closed-cell EPDM, 10 mm thick, self-adhesive | Airtight seal between HVAC and cabinet |
| Power connector | Industrial multipole (e.g., Harting Han or Wieland) | AC + 24V DC + PE in single connector |
| CAN connector | M12 4-pin, A-coded | CAN_H, CAN_L, GND, Shield |
| Condensate drain | 10 mm barb fitting with silicone hose | Routes condensate outside cabinet |
| Mounting bolts | M8×25 with captive nuts on cabinet side | 4× securing points |
| Alignment pins | 8 mm dowel pins (×2) | Ensures correct positioning and gasket compression |
| Quick-release latches | Spring-loaded (optional, for frequent service) | Tool-free removal for FRU swap |

### 8.3 Cabinet-Side Cutout

The main cabinet requires a prepared interface plate on the mounting face (side or rear wall) with matching air ports and connector pass-throughs.

```
CABINET WALL CUTOUT (External View)

    ┌────────────────────────────────────────────────────┐
    │                                                    │
    │   ┌──────────────────┐   ┌──────────────────┐      │
    │   │                  │   │                  │      │
    │   │   HOT AIR OUT    │   │   COLD AIR IN    │      │
    │   │   (to HVAC)      │   │   (from HVAC)    │      │
    │   │                  │   │                  │      │
    │   │   Flanged with   │   │   Internal       │      │
    │   │   weather drip   │   │   baffle to      │      │
    │   │   when no HVAC   │   │   plenum         │      │
    │   │   installed      │   │                  │      │
    │   └──────────────────┘   └──────────────────┘      │
    │                                                    │
    │   ○ Power    ○ CAN     ○ Drain                     │
    │   (bulkhead   (M12      (bulkhead                  │
    │    connector)  socket)   fitting)                  │
    │                                                    │
    │   ●       ●               ●       ●                │
    │  (M8 threaded inserts + alignment pin holes)       │
    │                                                    │
    └────────────────────────────────────────────────────┘

Note: Blanking plates cover all openings when HVAC unit
is removed for service, maintaining cabinet IP rating.
```

### 8.4 Mounting Options

```
SIDE MOUNT (Standard)         REAR MOUNT                   TOP MOUNT
┌──────┐ ┌──────┐            ┌──────┐                     ┌──────┐
│ MAIN │ │ HVAC │            │ HVAC │                     │ HVAC │
│ CAB  │ │ UNIT │            └──┬───┘                     └──┬───┘
│      │ │      │            ┌──┴───────────┐             ┌──┴───────────┐
│      │ │      │            │   MAIN CAB   │             │   MAIN CAB   │
└──────┘ └──────┘            └──────────────┘             └──────────────┘

Best for:                    Best for:                    Best for:
- Standard installation     - Wall-mounted units         - Limited side clearance
- Easy service access       - Narrow site constraints    - Indoor installations
- Most common config        - Rear access available      - Low-profile sites
```

## 9. Cold Climate Package (Optional)

For deployments where ambient temperature drops below -20°C, an optional cold-climate package is integrated into the HVAC unit.

### 9.1 Components

| Component | Specification | Purpose |
|-----------|---------------|---------|
| PTC Heater Element | 500W–1 kW, 230V AC | Maintains cabinet above 0°C during idle |
| Crankcase Heater | 30–50W, 230V AC | Keeps compressor oil warm for reliable starts |
| Defrost Controller | Integrated in HVAC controller firmware | Manages evaporator defrost cycles |
| Insulated Condensate Drain | Heated drain hose (self-regulating heat tape) | Prevents condensate freeze-up |
| Low-Ambient Fan Cycling | Condenser fan speed reduction / cycling | Maintains head pressure at low ambient |

### 9.2 Heating Mode Operation

```
HEATING MODE ACTIVATION: T1 (cabinet temp) < 5°C

1. Compressor OFF (not needed for heating)
2. PTC heater relay ON
3. Internal blower fan at 50% (circulate warm air)
4. External condenser fan OFF
5. PID control on T1 → cycle heater at setpoint (typically 10°C idle)

TRANSITION TO COOLING:
  When charging session starts and heat load appears:
  1. Heater OFF
  2. Wait 2 min for cabinet temp stabilization
  3. If T1 > setpoint → switch to COOLING mode
  4. If T1 still < setpoint → maintain VENTILATION + heater assist
```

### 9.3 Defrost Cycle

When the evaporator coil temperature (T3) drops below 0°C for more than 10 minutes during cooling operation, ice may form on the coil. The HVAC controller initiates a defrost cycle:

| Step | Duration | Action |
|------|----------|--------|
| 1 | 0 s | Close internal fan damper (or reduce fan to 10%) |
| 2 | 0 s | Activate hot-gas bypass valve (or reverse refrigerant flow) |
| 3 | 5–10 min | Hot gas heats evaporator coil above 10°C |
| 4 | — | Monitor T3 until >10°C confirmed |
| 5 | 0 s | Close hot-gas bypass, resume normal cooling |
| 6 | 30 s | Ramp internal fan back to normal speed |

Defrost is reported to Phytec SBC via operating mode (mode = DEFROST in CAN 0x101). The Phytec SBC may pre-emptively derate charger output during defrost if cabinet temperature is already elevated.

## 10. Maintenance and Serviceability

### 10.1 Routine Maintenance Schedule

| Interval | Task | Time |
|----------|------|------|
| Monthly | Visual inspection of condenser coil (debris, damage) | 5 min |
| Quarterly | Clean condenser coil (compressed air or water spray) | 15 min |
| Quarterly | Check condensate drain for blockage | 5 min |
| Semi-annually | Inspect internal evaporator coil and fan | 10 min |
| Annually | Check refrigerant pressure (high and low side) | 15 min |
| Annually | Verify CAN communication and sensor readings | 10 min |
| Annually | Inspect foam gaskets for compression set | 10 min |
| 3 years | Replace foam gaskets | 20 min |
| 5 years / as needed | Compressor replacement (FRU swap) | 2 hr |

### 10.2 FRU Swap Procedure

The entire HVAC unit is designed as a field-replaceable unit. Swap time target: **30 minutes** including functional verification.

```
HVAC FRU SWAP PROCEDURE

1. CHARGER → STANDBY (Phytec SBC sets HVAC mode = OFF via OCPP or local HMI)
   Wait for compressor to stop and pressures to equalize (~2 min)

2. DISCONNECT ELECTRICAL
   a. Unplug power connector (AC + DC)
   b. Unplug CAN M12 connector
   c. Disconnect condensate drain hose

3. REMOVE HVAC UNIT
   a. Remove 4× M8 securing bolts (13mm socket)
   b. Slide unit off alignment pins
   c. Lift unit away from cabinet (2-person lift for 350 kW variant)

4. INSTALL BLANKING PLATES (if not replacing immediately)
   a. Cover air ports with blanking plates
   b. Cap electrical connectors

5. INSTALL REPLACEMENT HVAC UNIT
   a. Verify gaskets are intact on new unit
   b. Align on dowel pins, push into position
   c. Tighten 4× M8 bolts (12 Nm, star pattern)
   d. Connect power, CAN, and drain

6. COMMISSION
   a. Power on charger → HVAC auto-detected on CAN #3
   b. Phytec SBC sends HVAC_Config (0x201) with alarm thresholds
   c. Phytec SBC sends mode = COOLING, verify compressor starts
   d. Verify temperatures trending correctly on HMI diagnostics
   e. Verify no CAN faults in safety supervisor log
```

### 10.3 Diagnostic Access

| Access Method | What It Provides |
|---------------|-----------------|
| CAN diagnostic messages (0x103) | Runtime hours, compressor cycles, energy consumption |
| Phytec SBC HMI diagnostic screen | Real-time temperatures, pressures, fan speeds, mode |
| OCPP `DiagnosticsStatusNotification` | Remote access to HVAC health data via backend |
| HVAC fault LED | Local visual indication of fault on the unit itself |
| UART debug port (on HVAC controller) | Low-level firmware diagnostics, sensor raw values |

## 11. Bill of Materials

### 11.1 Refrigeration Components

| Item | Qty | Specification | Notes |
|------|-----|---------------|-------|
| Scroll/rotary compressor (inverter-driven) | 1 | R-32, 9 kW (150 kW) / 20 kW (350 kW) | With rubber vibration mounts |
| Compressor inverter drive | 1 | 230V AC input, 3-phase output, variable frequency | Integrated or separate board |
| Condenser coil | 1 | Fin-and-tube, sized per variant | With protective guard |
| Evaporator coil | 1 | Fin-and-tube, copper/aluminum | Epoxy-coated fins |
| Electronic expansion valve (EEV) | 1 | Stepper motor driven, R-32 rated | With coil driver |
| Filter drier | 1 | Molecular sieve, 16 bar rated | On liquid line |
| Sight glass + moisture indicator | 1 | Brazed type | On liquid line |
| Service valves (Schrader) | 2 | High side + low side | For pressure check / charge |
| Copper refrigerant tubing | 1 set | 6.35 mm / 9.52 mm OD | Pre-bent or field brazed |

### 11.2 Fans and Airflow

| Item | Qty | Specification |
|------|-----|---------------|
| Internal centrifugal blower | 1 | 24V DC brushless, 500–3500 CFM (per variant), PWM |
| External axial condenser fan | 1 | 24V DC brushless, 800–2000 CFM, PWM, with rain guard |
| Condensate drip tray | 1 | Stainless steel or plastic, with drain fitting |
| Condensate drain hose | 1 | Silicone, 10 mm ID, 1.5 m length |

### 11.3 Sensors and Safety

| Item | Qty | Specification |
|------|-----|---------------|
| NTC 10kΩ temperature sensor | 4 | T1 (cabinet), T2 (condenser), T3 (evaporator), T4 (ambient) |
| Pressure transducer (high side) | 1 | 0–40 bar, 4–20 mA or 0.5–4.5V |
| High-pressure safety switch | 1 | NC, opens at 28 bar, manual reset |
| Low-pressure safety switch | 1 | NC, opens at 1.5 bar, auto-reset |
| Hall effect current sensor | 1 | 0–15A, for compressor current |

### 11.4 Electrical and Control

| Item | Qty | Specification |
|------|-----|---------------|
| HVAC controller PCB | 1 | STM32F103 or RP2350, with CAN transceiver |
| AC input EMI filter | 1 | Matched to compressor drive |
| MOV surge suppressor | 1 | 230V AC rated |
| Input fuse holder + fuse | 1 | gG, 16A or 32A per variant |
| PTC heater element (cold climate) | 1 | 500W–1 kW, 230V AC |
| Crankcase heater (cold climate) | 1 | 30–50W, 230V AC |
| Heater relay | 1 | 10A, 230V AC, coil 24V DC |

### 11.5 Mechanical and Interface

| Item | Qty | Specification |
|------|-----|---------------|
| HVAC enclosure | 1 | Powder-coated galv. steel, IP55 |
| Foam gaskets (EPDM, closed-cell) | 2 | 200×150 mm frame, 10 mm thick |
| Power connector (multipole) | 1 pair | Industrial, AC + DC + PE |
| CAN connector (M12) | 1 pair | 4-pin, A-coded, IP67 mated |
| Alignment dowel pins | 2 | 8 mm × 30 mm, stainless |
| M8 mounting bolts + captive nuts | 4 sets | Grade 8.8, with flat + spring washers |
| Vibration isolators (for compressor) | 4 | Rubber-metal, M8 |
| Condenser guard grille | 1 | Powder-coated steel, finger-safe |
| Blanking plate set (for service) | 1 | Covers air ports + connectors when HVAC removed |

## 12. Specifications Summary

| Parameter | 150 kW Variant | 350 kW Variant |
|-----------|----------------|----------------|
| Cooling capacity | 9 kW thermal | 20 kW thermal |
| Electrical input | ~3 kW (230V AC) | ~7 kW (230V AC) |
| COP | ~3.0 at 35°C ambient | ~2.8 at 35°C ambient |
| Refrigerant | R-32 (or R-290 for ≤3 kW units) | R-32 |
| Compressor type | Rotary / scroll, inverter-driven | Scroll, inverter-driven |
| Internal airflow | 500–1500 CFM | 1500–3500 CFM |
| Operating ambient range | -30°C to +50°C (with cold climate pkg) | -30°C to +50°C |
| Target cabinet temp | 30–35°C during charging | 30–35°C during charging |
| Power supply | 230V AC + 24V DC backup | 230V AC + 24V DC backup |
| CAN interface | CAN 2.0B, 250 kbps, M12 connector | CAN 2.0B, 250 kbps, M12 connector |
| IP rating (HVAC unit) | IP55 | IP55 |
| Dimensions (H×W×D) | 600×400×300 mm | 800×500×400 mm |
| Weight | 25–35 kg | 45–65 kg |
| FRU swap time | 30 min | 30 min (2-person) |
| Sound level | <55 dB(A) at 1 m | <60 dB(A) at 1 m |

## 13. References

- IEC 60335-2-40: Safety of household appliances — heat pumps and air conditioners
- IEC 60079-15: Equipment protection by type of protection "n" (if R-290)
- EN 378: Refrigerating systems and heat pumps — safety and environmental requirements
- [[01 - Hardware Components]] — Cabinet and system component specs
- [[03 - Cabinet Layout]] — Thermal zones and mounting positions
- [[04 - Backplane Power Management]] — PDU 3 cooling system power feed
- [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]] — Full CAN message dictionary
- [[docs/System/01 - System Architecture|01 - System Architecture]] — Thermal management architecture
- [[research/03 - 150kW DCFC Comparison|03 - 150kW DCFC Comparison]] — Design rationale for air cooling + HVAC

---

**Document Version**: 1.0
**Last Updated**: 2026-02-26
**Prepared by**: Mechanical & Thermal Engineering
