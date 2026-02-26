# Auxiliary Power Supply

Tags: #dcfc #components #power-supply #auxiliary #ups

Related: [[docs/Hardware/04 - Backplane Power Management|04 - Backplane Power Management]] | [[research/04 - Phytec SBC Replacement|04 - Phytec SBC Replacement]]

## 1. Overview

The auxiliary power supply subsystem generates the low-voltage DC rails (24V, 12V, 5V) that power all control electronics, sensors, communication modules, HMI, and cooling system. A redundant supply with battery backup ensures the charger can complete transactions gracefully during grid disturbances.

## 2. Components

| Item | Qty | Specification | Manufacturer | Model | Est. Price |
|------|-----|---------------|--------------|-------|------------|
| PSU 150W AC-DC (24V/10A) | 1 | Primary 24V supply, DIN rail | Meanwell | SDR-150-24 | $30 |
| PSU 150W AC-DC (24V/10A) | 1 | Redundant 24V supply (backup) | Meanwell | SDR-150-24 | $30 |
| PSU DC-DC (24V → 12V, 5V) | 1 | Dual-output DC-DC converter | Meanwell | DUPS40 | $30 |
| Diode-OR module | 1 | Redundancy switching for 24V bus | — | — | $20 (est.) |
| Lead-Acid Battery (12V, 10 Ah) | 2 | Maintenance-free, UPS backup | Exide | — | $40 |

## 3. Subtotal

| Category | Total |
|----------|-------|
| **Auxiliary Power Supply** | **$150** |

## 4. Notes

- Dual SMPS with diode-OR output provides N+1 redundancy — no single PSU failure takes down controls
- The two 12V LA batteries in series provide a 24V UPS bus (~10 minutes runtime at full aux load)
- The DUPS40 module manages battery charging and seamless switchover on AC loss
- 24V bus feeds PDU 3 (cooling) and PDU 4 (comms/HMI); 12V and 5V rails feed sensors, display, and modems
