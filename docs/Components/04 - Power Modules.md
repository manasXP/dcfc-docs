# Power Modules

Tags: #dcfc #components #power-electronics #power-module

Related: [[docs/Hardware/08 - Power Module Hardware Design|08 - Power Module Hardware Design]] | [[docs/Hardware/04 - Backplane Power Management|04 - Backplane Power Management]]

## 1. Overview

The power modules are the core energy conversion elements of the DCFC. Each 30 kW module (PDU-Micro) is a self-contained AC-to-DC converter (Vienna PFC + 3-phase interleaved DAB) using SiC MOSFETs. Modules are paralleled on a shared DC bus to scale total charger power. See the [[__Workspaces/PDU-Micro/__init|PDU-Micro]] workspace for the complete module design and [[__Workspaces/PDU-Micro/Components/07-Total Component Cost|PDU-Micro Total Component Cost]] for detailed BOM costing.

## 2. Components (Per Module)

### 2.1 Power Semiconductors

| Item | Qty | Specification | Reference |
|------|-----|---------------|-----------|
| SiC MOSFET 1200V/60A (PFC) | 6 | TO-247-4, 25–40 mΩ | Wolfspeed C3M0025120K |
| SiC MOSFET 1200V/40A (LLC) | 4 | TO-247-4, 40–60 mΩ | Wolfspeed C3M0040120K |
| SiC Schottky diode 1200V/80A (rectifier) | 4 | TO-247-3 | Wolfspeed C4D80120D |
| Isolated gate driver IC | 10 | ≥4A source, ≥8A sink, desat detect | Si8271 / ACPL-302J |

### 2.2 Magnetic Components

| Item | Qty | Specification |
|------|-----|---------------|
| PFC boost inductor | 3 | 200–400 µH, 50A, nanocrystalline core, Litz wire |
| LLC resonant inductor | 1 | 10–30 µH (may be integrated in transformer) |
| HF transformer | 1 | 25 kW, 80–300 kHz, ferrite/nanocrystalline, 4 kV isolation |
| CM choke (EMI filter) | 2 | 2–5 mH, nanocrystalline toroid |

### 2.3 Capacitors

| Item | Qty | Specification |
|------|-----|---------------|
| DC link film capacitors | 8–12 | 100–250 µF, 500V, polypropylene, low ESR |
| LLC resonant capacitor | 1–2 | 30–100 nF, 1200V, film |
| Output filter capacitor | 2–4 | 47–100 µF, 1200V, film |
| EMI filter X2 capacitors | 3 | 2.2 µF, 310V AC |
| EMI filter Y2 capacitors | 6 | 4.7 nF, 250V AC, ceramic safety rated |

### 2.4 Control and Sensing

| Item | Qty | Specification |
|------|-----|---------------|
| DSP/MCU | 1 | TMS320F280049C, LQFP-100 |
| CAN transceiver (isolated) | 1 | ISO 11898-2, 3 kV isolation |
| Current sensor (AC phases) | 3 | Rogowski coil or Hall, 60A range |
| Current sensor (DC output) | 1 | Closed-loop Hall, 80A range |
| NTC 10kΩ temperature sensors | 6 | MOSFETs, transformer, inductors, caps, PCB |
| Voltage dividers (isolated) | 3 | AC input, DC link, DC output |
| Optocoupler (ENABLE input) | 1 | 3 kV isolation |

### 2.5 Mechanical and Thermal

| Item | Qty | Specification |
|------|-----|---------------|
| Aluminum coldplate (machined) | 1 | 400×250×15 mm, serpentine channel |
| Thermal interface material (TIM) | 1 set | Pads or grease, 3–5 W/m·K |
| Module enclosure (top cover) | 1 | Sheet aluminum, powder-coated |
| Axial fan (80mm) | 1 | 24V DC, 50 CFM, PWM |
| Coolant fittings (10mm push-in) | 2 | Quick-disconnect |
| Input fuse holder + fuses | 1 set | 3-pole, 50A gG |
| AC terminal block | 1 | 4-position (L1, L2, L3, PE), 50A rated |
| DC terminal block | 1 | 2-position (DC+, DC-), 70A rated |
| CAN connectors | 2 | RJ45 or 4-pin Molex |
| ENABLE connector | 1 | 2-pin Molex or JST |
| Address DIP switch | 1 | 4-position |
| Status LED (RGB) | 1 | Panel-mount |
| M8 mounting bolts + alignment pins | 4+2 | For shelf mounting |

## 3. System-Level Quantity

| Configuration | Modules | Total Power |
|---------------|---------|-------------|
| 150 kW | 5 | 150 kW |
| 300 kW | 10 | 300 kW |

## 4. Cost Estimate

Per-module cost is based on the [[__Workspaces/PDU-Micro/Components/07-Total Component Cost|PDU-Micro BOM analysis]]:

| Item | @100 units | @500 units | Notes |
|------|-----------|-----------|-------|
| PDU-Micro module (30 kW, complete) | $1,806 | $1,436 | Per-module BOM incl. PCB fab + assembly |
| **5 modules (150 kW config)** | **$9,030** | **$7,180** | |
| Cabinet backplane + enclosure | $1,007 | $850 | Amortized across 5 modules |
| System CAN bus + wiring | $50 | $40 | |
| **150 kW System Total** | **$10,087** | **$8,070** | |

**Cost per kW:** $67/kW @100 units, $54/kW @500 units

## 5. Notes

- Module cost derived from PDU-Micro Architecture A design: open-frame sub-assembly (~440×320×120 mm, ~12 kg) installed in shared cabinet
- Top cost drivers: DAB SiC MOSFETs (23%), Vienna PFC module (13.3%), PCBs (13%)
- SiC semiconductors represent 36.3% of per-module BOM — Chinese SiC alternatives could save $100–130/module
- Each module is air-cooled within the cabinet (HVAC clip-on provides forced air); no per-module liquid cooling
- Modules communicate over CAN bus for current sharing (imbalance ≤5%) and receive voltage/current setpoints from the main controller
- The module DSP runs autonomous closed-loop CC/CV control; the main controller only sets high-level targets
