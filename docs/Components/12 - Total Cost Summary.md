# Total Cost Summary

Tags: #dcfc #components #cost #bom #budget

Related: [[docs/Components/01 - Control Electronics|01 - Control Electronics]] | [[docs/Components/04 - Power Modules|04 - Power Modules]] | [[docs/Components/11 - Cabinet and Enclosure|11 - Cabinet and Enclosure]]

## 1. Overview

This document provides a consolidated cost estimate for all major component groups of a 150 kW DC Fast Charger (single CCS2 connector) based on the detailed component lists in this folder. Power module costs are derived from the [[__Workspaces/PDU-Micro/Components/07-Total Component Cost|PDU-Micro BOM analysis]] at 100-unit volume. Other component prices are estimated from vendor research and published pricing as of early 2026.

## 2. Cost Breakdown by Subsystem

| # | Subsystem | Description | @100 (USD) | @500 (USD) |
|---|-----------|-------------|-----------|-----------|
| 1 | [[docs/Components/01 - Control Electronics\|Control Electronics]] | Phytec SBC, safety controller, IO controller, PLC modems | $730 | $730 |
| 2 | [[docs/Components/02 - Networking and Communication\|Networking and Communication]] | PoE switch, 5G modem, RFID/NFC | $375 | $375 |
| 3 | [[docs/Components/03 - User Interface\|User Interface]] | Touchscreen, E-Stop, status LEDs | $200 | $200 |
| 4 | [[docs/Components/04 - Power Modules\|Power Modules]] (5× 30 kW PDU-Micro) | SiC converters, PCBs, backplane, wiring | $10,087 | $8,070 |
| 5 | [[docs/Components/05 - DC Output Protection\|DC Output Protection]] | Contactors, breaker, fuses, sensors | $1,785 | $1,785 |
| 6 | [[docs/Components/06 - CCS Connector and Cable Assembly\|CCS Connector and Cable]] | Liquid-cooled CCS2 connector + 5 m cable | $2,180 | $2,180 |
| 7 | [[docs/Components/07 - AC Input and Protection\|AC Input and Protection]] | Breakers, contactors, SPD, meter, busbars, PDUs | $1,496 | $1,496 |
| 8 | [[docs/Components/08 - Auxiliary Power Supply\|Auxiliary Power Supply]] | SMPS (×2), DC-DC, UPS batteries | $150 | $150 |
| 9 | [[docs/Components/09 - Cooling System\|Cooling System]] | Pump, radiator, fans, sensors, coolant | $635 | $635 |
| 10 | [[docs/Components/10 - HVAC Clip-On Unit\|HVAC Clip-On Unit]] | Refrigeration, blowers, controller, enclosure | $1,720 | $1,720 |
| 11 | [[docs/Components/11 - Cabinet and Enclosure\|Cabinet and Enclosure]] | Main cabinet, mounting, cable management | $2,145 | $2,145 |
| | **TOTAL (150 kW, single connector)** | | **$21,503** | **$19,486** |

## 3. Cost Visualization (@100 units)

```
Power Modules (5×30 kW)      █████████████████████████████████  $10,087  (46.9%)
CCS Connector + Cable         ██████████                         $2,180   (10.1%)
Cabinet and Enclosure         ██████████                         $2,145   (10.0%)
DC Output Protection           ████████                          $1,785   (8.3%)
HVAC Clip-On Unit             ████████                          $1,720   (8.0%)
AC Input and Protection       ███████                           $1,496   (7.0%)
Control Electronics           ████                              $730     (3.4%)
Cooling System                ███                               $635     (3.0%)
Networking + Communication    ██                                $375     (1.7%)
User Interface                █                                 $200     (0.9%)
Auxiliary Power Supply        █                                 $150     (0.7%)
```

## 4. Power Module Cost Breakdown

From [[__Workspaces/PDU-Micro/Components/07-Total Component Cost|PDU-Micro BOM]]:

| Item | @100 | @500 |
|------|------|------|
| 5 × PDU-Micro modules (30 kW each) | $9,030 | $7,180 |
| Cabinet backplane + enclosure | $1,007 | $850 |
| System CAN bus + wiring | $50 | $40 |
| **Power Module Subsystem Total** | **$10,087** | **$8,070** |

Per-module: **$1,806 @100** / **$1,436 @500** — cost per kW: $67/kW @100, $54/kW @500

Top cost drivers per module: DAB SiC MOSFETs (23%), Vienna PFC module (13.3%), PCB fabrication (13%).

## 5. Configuration Variants

| Configuration | Change from Base | @100 (USD) | @500 (USD) |
|---------------|-----------------|-----------|-----------|
| 150 kW, single CCS2 connector | Base | **$21,503** | **$19,486** |
| 150 kW, dual connector (CCS2 + CCS2) | +1× cable assembly, +1× contactor set, +1× PLC | ~$25,700 | ~$23,700 |
| 300 kW, single connector | +5 power modules, larger breakers/busbars, larger HVAC | ~$32,900 | ~$28,900 |
| 350 kW, dual connector | Full configuration | ~$36,900 | ~$32,900 |

## 6. Cost Notes and Assumptions

### 6.1 What's Included
- All major electrical, electronic, and mechanical components
- Pre-assembled subassemblies (cable assembly, power modules, HVAC unit)
- Basic wiring, mounting hardware, and consumables
- PCB fabrication and SMD assembly for power modules

### 6.2 What's NOT Included
- **Assembly labor** — cabinet integration, wiring, testing, commissioning
- **Software licensing** — EVerest is open-source; OCPP backend may have licensing costs
- **Installation** — site preparation, concrete pad, utility connection, transformer
- **Shipping and logistics**
- **Certification and testing** — type testing, MID meter calibration, safety certification
- **Spare parts inventory**
- **Warranty reserve**

### 6.3 Pricing Assumptions
- Power module pricing is based on the PDU-Micro detailed BOM at 100-unit and 500-unit volumes (see [[__Workspaces/PDU-Micro/Components/07-Total Component Cost|PDU-Micro Total Component Cost]])
- Non-module components priced at small-volume rates (1–10 units); volume discounts expected at higher quantities
- Prices in USD, sourced from DigiKey, Robu, RoboCraze, Microchip Direct, and vendor quotations
- 5G modem price ($200) is an estimate pending carrier and module selection
- Chinese SiC alternates could reduce per-module cost by $100–130 (see PDU-Micro cost optimization notes)

### 6.4 Comparison to Market
- Commercial 150 kW DC fast chargers retail for $30,000–$60,000 depending on brand, features, and region
- The estimated BOM cost of ~$21,500 @100 (without labor, installation, or margin) represents strong cost competitiveness
- At 500-unit volume (~$19,500), the BOM supports a competitive retail price point with healthy margins

## 7. References

- [[__Workspaces/PDU-Micro/Components/07-Total Component Cost|PDU-Micro Total Component Cost]] — Detailed per-module BOM and costing
- [[__Workspaces/PDU-Micro/__init|PDU-Micro]] — 30 kW power module design specification
- [[research/04 - Phytec SBC Replacement|04 - Phytec SBC Replacement]] — Main controller selection and BOM impact
- [[docs/Hardware/05 - DC Output Contactor and Pre-Charge Circuit|05 - DC Output Contactor and Pre-Charge Circuit]] — Contactor assembly BOM
- [[docs/Hardware/07 - CCS Connector and Liquid-Cooled Cable Assembly|07 - CCS Connector and Liquid-Cooled Cable Assembly]] — Cable assembly BOM
- [[docs/Hardware/08 - Power Module Hardware Design|08 - Power Module Hardware Design]] — Power module hardware design
- [[docs/Hardware/06 - HVAC Clip-On Unit Hardware Design|06 - HVAC Clip-On Unit Hardware Design]] — HVAC unit BOM
- [[docs/Hardware/04 - Backplane Power Management|04 - Backplane Power Management]] — PDU and busbar BOM
