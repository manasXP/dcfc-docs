# DC Output Protection

Tags: #dcfc #components #protection #contactor #safety

Related: [[docs/Hardware/05 - DC Output Contactor and Pre-Charge Circuit|05 - DC Output Contactor and Pre-Charge Circuit]]

## 1. Overview

The DC output protection assembly provides the switching, overcurrent protection, measurement, and pre-charge circuitry between the power module DC bus and the EV charging connector. All components are controlled by the safety supervisor (STM32), not the main controller.

## 2. Components

| Item | Qty | Specification | Reference | Est. Price |
|------|-----|---------------|-----------|------------|
| DC contactor 500A/1000V DC | 2 | K2, K3 — main DC+/DC-, with 2NO+2NC aux | Gigavac GX26 / TE EV200 | $600 |
| DC contactor 50A/1000V DC | 1 | K4 — pre-charge, with 1NO aux | Gigavac GX14 / TE EV100 | $150 |
| Pre-charge resistor 100Ω/100W | 1 | R1 — wirewound, 1500J pulse rating | Vishay RPS / Ohmite | $30 |
| Discharge resistor 10kΩ/100W | 1 | R2 — wirewound, 1200V rated | Vishay RH series | $25 |
| DC circuit breaker 500A/1000V | 1 | CB1 — thermal-magnetic, with aux contacts | ABB Tmax / Schneider NSX | $400 |
| Semiconductor fuse 500A/1200V gR | 2 | F1, F2 — ultra-fast, bolt-in | Bussmann FWH / Mersen | $200 |
| Fuse holder (bolt-in) | 2 | Matched to fuse type | — | $60 |
| Hall effect current sensor ±600A | 1 | Closed-loop, 4–20 mA output | LEM DHAB / Honeywell | $80 |
| Voltage sensor module 0–1200V | 1 | Isolated, 0–10V output | LEM LV25-P | $50 |
| Flyback TVS diodes | 3 | For K2, K3, K4 coils, 48V clamp | — | $10 |
| MOSFET coil driver boards | 3 | Isolated gate drive, PWM capable | Custom / off-shelf | $60 |
| Terminal block TB2 (DC output) | 1 | 3-position, 95 mm² rated, 1000V | Phoenix Contact PTPOWER | $40 |
| Mounting hardware | 1 set | M8, M6, M5 bolts, Belleville washers, Grade 8.8 | — | $30 |
| Busbar links (internal) | 4 | 95 mm² rated copper, tin-plated | Custom cut | $50 |

## 3. Subtotal

| Category | Total |
|----------|-------|
| **DC Output Protection** | **$1,785** |

## 4. Notes

- K2 and K3 are 500A-rated main DC contactors with PWM hold coil drive to reduce steady-state heat
- K4 pre-charge contactor limits inrush to 10A max at 1000V through R1 (100Ω)
- Protection hierarchy: Safety supervisor electronic OCP → DC breaker CB1 → Semiconductor fuses F1/F2
- The discharge resistor R2 bleeds output voltage to <60V within 30 seconds per IEC 61851-23
- Contactor weld detection runs after every open command via auxiliary feedback contacts
