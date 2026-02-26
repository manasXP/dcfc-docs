# HVAC Clip-On Unit

Tags: #dcfc #components #hvac #cooling #thermal

Related: [[docs/Hardware/06 - HVAC Clip-On Unit Hardware Design|06 - HVAC Clip-On Unit Hardware Design]] | [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]]

## 1. Overview

The HVAC clip-on unit is a self-contained, externally mounted refrigeration system that manages the internal cabinet air temperature. It creates a closed-loop internal air circuit (sealed cabinet, IP55+) and is a field-replaceable unit (FRU). The unit communicates with the main controller via CAN bus.

## 2. Components

### 2.1 Refrigeration

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| Scroll/rotary compressor (inverter-driven) | 1 | R-32, 9 kW (150 kW variant) | $500 |
| Compressor inverter drive | 1 | 230V AC input, variable frequency | $200 |
| Condenser coil (fin-and-tube) | 1 | With protective guard | $100 |
| Evaporator coil (fin-and-tube) | 1 | Copper/aluminum, epoxy-coated | $100 |
| Electronic expansion valve (EEV) | 1 | Stepper motor driven, R-32 rated | $80 |
| Filter drier | 1 | Molecular sieve, 16 bar | $20 |
| Sight glass + moisture indicator | 1 | Brazed type | $15 |
| Service valves (Schrader) | 2 | High side + low side | $10 |
| Copper refrigerant tubing | 1 set | 6.35 / 9.52 mm OD, pre-bent | $30 |

### 2.2 Fans and Airflow

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| Internal centrifugal blower | 1 | 24V DC brushless, PWM | $80 |
| External axial condenser fan | 1 | 24V DC brushless, PWM, rain guard | $60 |
| Condensate drip tray | 1 | Stainless steel, with drain fitting | $15 |
| Condensate drain hose | 1 | Silicone, 10 mm ID, 1.5 m | $5 |

### 2.3 Sensors and Safety

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| NTC 10kΩ temperature sensors | 4 | Cabinet, condenser, evaporator, ambient | $20 |
| Pressure transducer (high side) | 1 | 0–40 bar, 4–20 mA | $40 |
| High-pressure safety switch | 1 | NC, opens at 28 bar, manual reset | $20 |
| Low-pressure safety switch | 1 | NC, opens at 1.5 bar, auto-reset | $15 |
| Hall effect current sensor | 1 | 0–15A, compressor current | $15 |

### 2.4 Electrical and Control

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| HVAC controller PCB | 1 | STM32F103 or RP2350, CAN transceiver | $40 |
| AC input EMI filter | 1 | Matched to compressor drive | $20 |
| MOV surge suppressor | 1 | 230V AC rated | $5 |
| Input fuse holder + fuse | 1 | gG, 16A | $10 |
| PTC heater element (cold climate) | 1 | 500W–1 kW, 230V AC | $30 |
| Crankcase heater | 1 | 30–50W, 230V AC | $15 |
| Heater relay | 1 | 10A, 230V AC, coil 24V DC | $10 |

### 2.5 Mechanical and Interface

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| HVAC enclosure | 1 | Powder-coated galv. steel, IP55 | $150 |
| Foam gaskets (EPDM, closed-cell) | 2 | 200×150 mm, 10 mm thick | $10 |
| Power connector (multipole) | 1 pair | Industrial, AC + DC + PE | $30 |
| CAN connector (M12) | 1 pair | 4-pin, A-coded, IP67 | $20 |
| Alignment dowel pins | 2 | 8 mm × 30 mm, stainless | $5 |
| M8 mounting bolts + captive nuts | 4 sets | Grade 8.8, flat + spring washers | $10 |
| Vibration isolators (compressor) | 4 | Rubber-metal, M8 | $15 |
| Condenser guard grille | 1 | Powder-coated steel, finger-safe | $20 |
| Blanking plate set | 1 | Covers ports when HVAC removed | $15 |

## 3. Subtotal

| Category | Total |
|----------|-------|
| **HVAC Clip-On Unit** | **$1,720** |

## 4. Notes

- The HVAC unit is designed as a field-replaceable module — it can be swapped without opening the main cabinet
- Cooling capacity: ~9 kW (150 kW charger) including solar load and ambient ingress with 20% margin
- CAN bus interface allows the main controller to monitor and control compressor speed, fan speed, and heater state
- Cold-climate operation supported via PTC heater and crankcase heater for temperatures down to -30°C
