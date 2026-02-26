# Cooling System

Tags: #dcfc #components #cooling #thermal #liquid-cooling

Related: [[docs/Hardware/01 - Hardware Components|01 - Hardware Components]] | [[docs/Hardware/03 - Cabinet Layout|03 - Cabinet Layout]]

## 1. Overview

The cooling system manages thermal loads from the power modules, contactors, and busbars via a liquid coolant loop. The loop circulates glycol-water mixture through the power module coldplates and the CCS connector/cable, rejecting heat through a radiator with forced-air fans.

## 2. Components

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| Coolant pump (24V DC brushless) | 1 | 10–30 L/min, PWM speed control | $150 |
| Coolant reservoir (2 L) | 1 | With level sensor | $40 |
| Radiator / heat exchanger | 1 | Air-to-liquid, sized for 10 kW heat rejection | $200 |
| Radiator axial fan (24V) | 2 | PWM, 24V DC, 2A each | $60 |
| Flow sensor | 1 | 1–50 L/min, pulse output | $40 |
| NTC temperature sensors | 4 | Coolant inlet, outlet, ambient, cabinet | $20 |
| Coolant manifold / distribution block | 1 | For parallel module feeds | $50 |
| Coolant tubing (silicone, 10 mm ID) | 5 m | Rated to 100°C, 5 bar | $30 |
| Hose clamps / quick-disconnects | 12 | Assorted sizes | $30 |
| Coolant (50/50 ethylene glycol-water) | 5 L | Pre-mixed | $15 |

## 3. Subtotal

| Category | Total |
|----------|-------|
| **Cooling System** | **$635** |

## 4. Notes

- The coolant loop serves power module coldplates (3–5 L/min per module) and the liquid-cooled CCS cable
- Total flow rate: 10–30 L/min depending on power level and number of active modules
- Target coolant temperature: 30–45°C inlet; max 60°C outlet
- Heat dissipation: ~1 kW per 25 kW module (at 96% efficiency) = ~6 kW for 150 kW configuration
- Temperature sensors on coolant lines provide input to the EVerest HVAC driver for thermal management
