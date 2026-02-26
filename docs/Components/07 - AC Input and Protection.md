# AC Input and Protection

Tags: #dcfc #components #ac-input #protection #breaker

Related: [[docs/Hardware/04 - Backplane Power Management|04 - Backplane Power Management]] | [[docs/Hardware/03 - Cabinet Layout|03 - Cabinet Layout]]

## 1. Overview

The AC input and protection stage provides the interface between the utility grid and the charger's internal power distribution. It includes the main disconnect, overcurrent protection, surge suppression, energy metering, and the backplane busbar system that distributes AC power to all PDUs.

## 2. Components

### 2.1 Main Input Protection

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| Main disconnect switch (350A, 3-pole) | 1 | Lockable, door-interlocked | $120 |
| AC contactor K1 (400A, 3-pole) | 1 | 24V DC coil, via safety relay chain | $250 |
| Surge protection device (SPD Type 2) | 1 | 3-phase + N, 40 kA | $80 |
| Energy meter (MID certified) | 1 | Class 0.5, Modbus RTU/TCP | $150 |
| EMI filter | 1 | 3-phase, matched to system rating | $80 |

### 2.2 Backplane Busbar Assembly

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| Copper busbar L1, L2, L3 (30×5 mm, 700 mm) | 3 | Tin-plated C110 copper, 400A rated | $90 |
| Copper busbar N (20×5 mm, 700 mm) | 1 | Tin-plated | $25 |
| Copper busbar PE (30×5 mm, 700 mm) | 1 | Bonded to chassis | $30 |
| Insulated standoffs (M8, 10 kV) | 20 | Polyester | $40 |
| Phase barrier plates | 4 | Polycarbonate, 700×30 mm | $20 |
| Busbar covers (IP2X finger-safe) | 5 | Polycarbonate | $30 |
| Tap connectors (bolted) | 16 | M8 brass bolts, Belleville washers | $25 |

### 2.3 PDU Branch Protection

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| 3-pole MCB 175A Type C (PDU 1) | 2 | Power module branch breakers | $120 |
| 3-pole AC contactor 200A (PDU 1) | 2 | Power module feed contactors | $200 |
| 1-pole MCB 16A Type C (PDU 2) | 2 | SMPS branch breakers | $20 |
| Current transformers 200A/5A | 6 | 3 per power module branch | $60 |
| Automotive blade fuses (assorted) | 12 | PDU 3 and PDU 4 | $15 |
| DIN-rail fuse holders | 12 | DIN rail mount | $30 |
| DIN rail (35 mm) | 4 m | PDU mounting | $15 |
| Terminal blocks (assorted) | 40 | DIN rail mount | $40 |

### 2.4 DIN Rail Components

| Item | Qty | Specification | Est. Price |
|------|-----|---------------|------------|
| MCB 10A (24V DC supply) | 2 | Primary + backup | $12 |
| MCB 6A (control + comms) | 2 | — | $12 |
| MCB 4A (fan + pump) | 2 | — | $12 |
| Safety relay module | 1 | Dual-channel, category 3 | $80 |
| Relay bank (K1–K4 + 2 spare) | 6 | DIN rail relay sockets | $60 |

## 3. Subtotal

| Category | Total |
|----------|-------|
| **AC Input and Protection** | **$1,496** |

## 4. Notes

- The main disconnect switch is door-interlocked — opening the cabinet door de-energizes high voltage
- Protection coordination ensures PDU branch breakers trip before the main breaker (selectivity)
- The energy meter is MID-certified for billing-grade accuracy (Class 0.5)
- The busbar system handles up to 400A continuous across all three phases
