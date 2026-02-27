# Power Module Hardware Design

Tags: #dcfc #hardware #power-electronics #sic #pfc #llc #power-module

Related: [[01 - Hardware Components]] | [[02 - Electric Wiring Diagram]] | [[03 - Cabinet Layout]] | [[04 - Backplane Power Management]] | [[docs/System/01 - System Architecture|01 - System Architecture]] | [[docs/Software/04 - Power Module CAN Bus Interface|04 - Power Module CAN Bus Interface]]

## 1. Overview

The power module is the core energy conversion element of the DCFC. Each module is a self-contained 30 kW AC-to-DC converter that takes 3-phase AC grid power and produces a regulated, isolated DC output suitable for EV battery charging. Multiple modules are connected in parallel on a shared DC output bus to scale the charger's total power — five modules for 150 kW, ten for 300 kW.

Each module contains two conversion stages:
1. **PFC Stage** (AC → DC link) — A Vienna rectifier that converts 3-phase AC to a stable ~800V DC link while maintaining >0.99 power factor and <5% THD
2. **DC-DC Stage** (DC link → Isolated DC output) — An LLC resonant converter with a high-frequency transformer that provides galvanic isolation and regulates the output voltage across the full 200–1000V range

The module includes its own DSP controller, gate drivers, sensors, protection circuits, and CAN interface. It operates autonomously in closed-loop control and receives high-level voltage/current setpoints from the Phytec SBC main controller over CAN #1.

## 2. Module Block Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           30 kW POWER MODULE                                     │
│                                                                                  │
│  AC INPUT                                                              DC OUTPUT │
│  3-Phase    ┌─────┐   ┌──────────────────────┐   ┌─────┐              200-1000V │
│  400-480V ─►│ EMI │──►│    PFC STAGE          │──►│ DC  │                        │
│  50/60Hz    │Filtr│   │    (Vienna Rectifier)  │   │Link │   ┌───────────────┐   │
│             └─────┘   │                        │   │     │   │   DC-DC STAGE │   │
│                       │  ┌─────┐┌─────┐┌─────┐│   │     │──►│   (LLC        │──►│
│                       │  │Boost││Boost││Boost││   │800V │   │   Resonant)   │   │
│                       │  │ L1  ││ L2  ││ L3  ││   │     │   │               │   │
│                       │  └──┬──┘└──┬──┘└──┬──┘│   │     │   │  ┌────────┐   │   │
│                       │     │      │      │   │   │     │   │  │ HF     │   │   │
│                       │  ┌──┴──────┴──────┴──┐│   │ Cap │   │  │Xformer │   │   │
│                       │  │  SiC MOSFET       ││   │ Bank│   │  │(Isol.) │   │   │
│                       │  │  Bridge (6-switch) ││   │     │   │  └────────┘   │   │
│                       │  └───────────────────┘│   │     │   │               │   │
│                       └──────────────────────┘   └─────┘   └───────────────┘   │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                        MODULE CONTROLLER                                    │ │
│  │                                                                             │ │
│  │  ┌──────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────┐ ┌──────────────┐ │ │
│  │  │ DSP  │ │  Gate    │ │ Voltage  │ │ Current  │ │Temp │ │   CAN #1     │ │ │
│  │  │ MCU  │◄│  Drivers │ │ Sensing  │ │ Sensing  │ │Sens │ │  Interface   │ │ │
│  │  │      │►│ (6× PFC  │ │ (AC in,  │ │ (AC in,  │ │(6×  │ │  to Phytec SBC     │ │ │
│  │  │      │ │  4× LLC) │ │  DC link,│ │  DC out) │ │NTC) │ │  (500 kbps) │ │ │
│  │  └──────┘ └──────────┘ │  DC out) │ └──────────┘ └─────┘ └──────────────┘ │ │
│  │                        └──────────┘                                         │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                      │ │
│  │  │   ENABLE     │  │  Auxiliary   │  │  Status      │                      │ │
│  │  │   Hardware   │  │  Power      │  │  LED / Fan   │                      │ │
│  │  │   Input      │  │  Supply     │  │  Control     │                      │ │
│  │  │ (from Safety │  │  (24V→15V,  │  │              │                      │ │
│  │  │  Supervisor) │  │   5V, 3.3V) │  │              │                      │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                      │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  ┌──────────────────┐   ┌──────────────────┐                                    │
│  │  Coolant Inlet   │   │  Coolant Outlet  │  (Liquid cooling for heatsink)     │
│  └──────────────────┘   └──────────────────┘                                    │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## 3. Electrical Specifications

### 3.1 Input Specifications

| Parameter | Value | Notes |
|-----------|-------|-------|
| Input voltage | 3-phase 400–480V AC ±10% | 360–528V operating range |
| Frequency | 50/60 Hz ±5% | Auto-detecting |
| Input current (per phase) | 52A RMS at 400V / 42A RMS at 480V | At 30 kW full load |
| Power factor | >0.99 at >50% load | Vienna PFC topology |
| THD (current) | <5% at rated load | IEC 61000-3-12 compliant |
| Inrush current | <20A peak (soft-start via PFC) | NTC + relay bypass |
| Input fusing | Internal 63A gG per phase | 3-pole fuse holder |

### 3.2 Output Specifications

| Parameter | Value | Notes |
|-----------|-------|-------|
| Output voltage range | 200–1000V DC | Continuous adjustment via LLC |
| Maximum output current | 75A at 400V / 30A at 1000V | Power-limited to 30 kW |
| Rated output power | 30 kW | Continuous at all voltages within P=V×I envelope |
| Voltage regulation accuracy | ±0.5% of setpoint | Under steady-state load |
| Current regulation accuracy | ±1% of setpoint | Under steady-state operation |
| Voltage ripple (peak-to-peak) | <1% of V_out | At rated load, 200 kHz bandwidth |
| Current ripple | <5% of I_out | At rated load |
| Transient response | <5% overshoot, <10 ms settling | 25% to 100% load step |
| Output isolation | 4 kV DC (input to output) | Per IEC 62477-1 |
| Parallel operation | Up to 12 modules on shared DC bus | Droop-mode current sharing |

### 3.3 Power-Voltage Envelope

```
Output Current (A)
│
│ 150 ┤ ........
│     │ .       .
│     │ .        .         POWER LIMIT BOUNDARY
│ 125 ┤ .         .        P = 30 kW
│     │ .          .
│ 100 ┤ .           .
│     │ .            .
│  75 ┤ .             .......
│     │ .                    .
│  50 ┤ .                     .
│     │ .                      .
│  25 ┤ .                       ........
│     │ .                              .
│   0 ┤─┴──────┬──────┬──────┬──────┬──┴──
│     200    400    600    800   1000
│                Output Voltage (V)

At 200V: I_max = 30000/200 = 150A (but capped at 75A by module rating)
At 400V: I_max = 30000/400 = 75.0A (module max current)
At 600V: I_max = 30000/600 = 50.0A
At 800V: I_max = 30000/800 = 37.5A
At 1000V: I_max = 30000/1000 = 30.0A

NOTE: Below 400V, output is current-limited to 75A,
      so actual power = V × 75A (e.g., 200V × 75A = 15 kW)
```

### 3.4 Efficiency

| Load | Efficiency | Notes |
|------|-----------|-------|
| 10% (3 kW) | >92% | Switching losses dominate |
| 25% (7.5 kW) | >95% | |
| 50% (15 kW) | >96.5% | Near-optimal operating point |
| 75% (22.5 kW) | >97% | Peak efficiency zone |
| 100% (30 kW) | >96% | Conduction losses increase |

Peak efficiency of ~97% occurs at 70–80% load, which is typical during the constant-current phase of EV charging. The module is optimized for this operating point.

**Loss breakdown at 30 kW (96% efficiency = 1.2 kW losses):**

| Loss Source | Estimated Loss | Percentage |
|-------------|---------------|------------|
| PFC SiC MOSFET switching | 240W | 20% |
| PFC SiC MOSFET conduction | 120W | 10% |
| PFC boost inductor core + copper | 180W | 15% |
| DC-DC SiC MOSFET switching | 180W | 15% |
| DC-DC SiC MOSFET conduction | 96W | 8% |
| HF transformer core + copper | 180W | 15% |
| Output rectifier conduction | 120W | 10% |
| Gate drivers + control circuits | 36W | 3% |
| EMI filter + misc | 48W | 4% |
| **Total** | **~1200W** | **100%** |

## 4. PFC Stage — Vienna Rectifier

### 4.1 Topology

The Vienna rectifier is a three-level, three-phase boost PFC topology that achieves high power factor, low THD, and low switch stress using only six active switches (two per phase) rather than twelve in a conventional active bridge.

```
            L1 (AC Phase A)
               │
               │   ┌─────────────┐
               ├───┤ Boost       │
               │   │ Inductor LA │
               │   └──────┬──────┘
               │          │
               │     ┌────┴────┐
               │     │         │
               │   ┌─┴─┐   ┌──┴──┐
               │   │Q1a│   │Q1b  │    SiC MOSFETs
               │   │   │   │     │    (bidirectional switch)
               │   └─┬─┘   └──┬──┘
               │     │        │
               │     └────┬───┘
               │          │
               │     ┌────┴────┐
               │     │ Midpoint│───────── DC Link Midpoint (0V)
               │     └─────────┘
               │
          (Same topology repeated for Phase B and Phase C)

    DC Link:
        DC+ ────── C_upper ────── Midpoint ────── C_lower ────── DC-
        (+800V)    (1000µF)        (0V)           (1000µF)       (0V)

    Total DC Link Voltage: ~800V (at 400V AC input)
    DC Link ripple: ~24V peak-to-peak at 30 kW
```

### 4.2 Vienna Rectifier Advantages

| Advantage | Explanation |
|-----------|-------------|
| Only 6 active switches | Half the switches of a full active bridge — lower cost, simpler gate drive |
| Three-level operation | Reduced voltage stress on switches (V_dc/2 per switch) |
| Inherent boost PFC | Power factor >0.99 without auxiliary PFC stage |
| Low THD | <5% current THD with simple control |
| Unidirectional power flow | Suitable for charger applications (no energy return to grid) |
| Reduced EMI | Three-level switching produces lower dv/dt |

### 4.3 PFC Switching Components

#### SiC MOSFETs (PFC Stage)

| Parameter | Value | Notes |
|-----------|-------|-------|
| Quantity | 6 (2 per phase, bidirectional switch pair) | TO-247-4 or D²PAK-7L |
| Voltage rating | 1200V | Handles DC link voltage + margin |
| Current rating | 60A continuous | Per device, at 100°C case |
| R_DS(on) | 25–40 mΩ at 25°C | SiC advantage: low conduction loss |
| Switching frequency | 65 kHz | Balance of efficiency and inductor size |
| Gate voltage | +18V / -5V | Per SiC gate driver specification |
| Rise time | <20 ns | Fast switching for low losses |
| Device type | e.g., Wolfspeed C3M0025120K, Infineon IMZ120R040M1H, or Rohm SCT3040AR | 1200V SiC MOSFET |

#### Boost Inductors

| Parameter | Value | Notes |
|-----------|-------|-------|
| Quantity | 3 (one per phase) | Three separate inductors |
| Inductance | 200–400 µH | Tuned for 65 kHz, ripple <20% of I_peak |
| Saturation current | >50A | Must not saturate at peak load |
| Core material | Amorphous or nanocrystalline (e.g., Metglas, Vitroperm) | Low core loss at 65 kHz |
| Winding | Litz wire (to reduce skin/proximity effect) | 60–100 strand, 0.2 mm wire |
| Temperature rating | 130°C (Class B) | With thermal pad to heatsink |
| Dimensions | ~80×60×50 mm each | Custom wound or standard COTS |

#### DC Link Capacitors

| Parameter | Value | Notes |
|-----------|-------|-------|
| Total capacitance | 2000 µF (2× 1000 µF in series for split DC link) | Upper bank + lower bank |
| Voltage rating | 500V per bank (1000V total across both) | With 20% margin above 800V link |
| Type | Film capacitors (polypropylene) | Superior ripple current vs electrolytic |
| Ripple current rating | >30A RMS per bank | At 65 kHz |
| ESR | <10 mΩ per bank | Low losses |
| Lifetime | >100,000 hours at 85°C | Derated for 60°C operating |
| Temperature rating | 105°C max case | Monitored by NTC sensor |
| Configuration | 4–6 capacitors per bank in parallel | To achieve capacitance and ripple rating |

### 4.4 PFC Control

The PFC stage uses dual-loop control: an outer voltage loop regulates the DC link voltage (800V target), and an inner current loop shapes each phase current to be sinusoidal and in-phase with the grid voltage.

```
                    ┌───────────────────────────────────────────┐
                    │         PFC CONTROL (per phase)           │
                    │                                           │
  V_dc_ref ──►(+)──┼──►┌──────────┐   ┌──────────┐            │
  (800V)      (-)  │   │ Voltage  │──►│ Current  │──► PWM ──► Gate
               │   │   │ PI Loop  │   │ PI Loop  │     to Q1a/Q1b
  V_dc_meas ──┘   │   │ (slow)   │   │ (fast)   │            │
                    │   └──────────┘   └────┬─────┘            │
                    │        │              │                   │
                    │   I_ref (sinusoidal   │                   │
                    │    envelope)     I_L_meas                 │
                    │        │              │                   │
                    │   ┌────┴────┐    ┌────┴────┐             │
                    │   │  PLL   │    │ Current │             │
                    │   │(phase  │    │ Sensor  │             │
                    │   │ lock)  │    │  (LA)   │             │
                    │   └────┬───┘    └─────────┘             │
                    │        │                                 │
                    │   V_grid (phase angle)                   │
                    │                                          │
                    └──────────────────────────────────────────┘

Voltage loop bandwidth: ~20 Hz (slow, rejects 100/120 Hz ripple)
Current loop bandwidth: ~5 kHz (fast, tracks sinusoidal reference)
Switching frequency: 65 kHz (fixed)
Modulation: Carrier-based PWM or SVM (Space Vector Modulation)
```

## 5. DC-DC Stage — LLC Resonant Converter

### 5.1 Topology

The LLC resonant converter is a half-bridge or full-bridge topology that achieves zero-voltage switching (ZVS) across a wide load range, resulting in very low switching losses and high efficiency. The resonant tank (L_r, C_r, L_m) shapes the current waveform to be sinusoidal, and the high-frequency transformer provides galvanic isolation.

```
    DC Link (800V)
        │
   ┌────┴────┐
   │         │
   │  ┌───┐  │   ┌───────────────────────────────────────────────┐
   │  │Q3 │  │   │              RESONANT TANK                    │
   │  └─┬─┘  │   │                                               │
   │    │    │   │   ┌────┐    ┌────┐    ┌──────────────────┐    │
   │    ├────┼───┼──►│ Lr │───►│ Cr │───►│  HF TRANSFORMER  │    │
   │    │    │   │   │    │    │    │    │                  │    │
   │  ┌─┴─┐  │   │   └────┘    └────┘    │  ┌────┐  ┌────┐ │    │
   │  │Q4 │  │   │                        │  │ Np │  │ Ns │ │    │    DC OUT
   │  └─┬─┘  │   │   Lm (magnetizing)    │  │    │  │    │─┼────┼──► 200-1000V
   │    │    │   │   ┌────┐               │  └────┘  └────┘ │    │
   │    │    │   │   │ Lm │ (in parallel  │                  │    │
   │    │    │   │   └────┘  with Xformer)│  Turns ratio:    │    │
   │    │    │   │                        │  Np:Ns = varies  │    │
   │    │    │   └────────────────────────┴──────────────────┘    │
   │    │    │                                                    │
   └────┴────┘                                                    │
                                                                  │
                                            ┌─────────────────────┘
                                            │
                                  ┌─────────┴─────────┐
                                  │  OUTPUT RECTIFIER  │
                                  │  (SiC Schottky     │
                                  │   diodes or        │
                                  │   synchronous      │
                                  │   SiC MOSFETs)     │
                                  │                    │
                                  │  ┌────┐  ┌────┐   │
                                  │  │ D1 │  │ D2 │   │    ┌─────┐
                                  │  └─┬──┘  └──┬─┘   │───►│C_out│──► DC+
                                  │    │        │     │    └─────┘
                                  │    └───┬────┘     │
                                  │        │          │              DC-
                                  │       GND         │──────────────►
                                  └───────────────────┘
```

### 5.2 LLC Resonant Advantages

| Advantage | Explanation |
|-----------|-------------|
| Zero-voltage switching (ZVS) | Primary MOSFETs switch at zero voltage — near-zero switching loss |
| Sinusoidal resonant current | Low EMI, reduced stress on transformer |
| Galvanic isolation | HF transformer provides 4 kV input-to-output isolation |
| Variable frequency control | Output voltage regulated by varying switching frequency |
| Wide output voltage range | 200–1000V achievable with appropriate transformer ratio |
| High efficiency at partial load | ZVS maintained across wide load range |

### 5.3 LLC Switching Components

#### SiC MOSFETs (LLC Primary)

| Parameter | Value | Notes |
|-----------|-------|-------|
| Quantity | 4 (full-bridge configuration) | TO-247-4 or D²PAK-7L |
| Voltage rating | 1200V | DC link voltage + margin |
| Current rating | 40A continuous | Lower current than PFC stage |
| R_DS(on) | 40–60 mΩ | Acceptable due to ZVS |
| Switching frequency | 80–300 kHz (variable) | Below resonance = voltage boost, above = voltage buck |
| Gate voltage | +18V / -5V | Shared gate driver design with PFC |

#### Resonant Components

| Component | Value | Notes |
|-----------|-------|-------|
| L_r (resonant inductance) | 10–30 µH | Can be integrated into transformer leakage |
| C_r (resonant capacitance) | 30–100 nF | Film capacitor, low ESR, high ripple current |
| L_m (magnetizing inductance) | 100–300 µH | Part of transformer design |
| Resonant frequency (f_r) | ~100 kHz | f_r = 1 / (2π√(L_r × C_r)) |
| Operating frequency range | 80–300 kHz | Below f_r for boost, above for buck |

#### High-Frequency Transformer

| Parameter | Value | Notes |
|-----------|-------|-------|
| Power rating | 30 kW | With derating for core temperature |
| Operating frequency | 80–300 kHz | Designed for resonant frequency |
| Turns ratio (Np:Ns) | Optimized for 800V input → 200–1000V output | Typically ~1:1 to 1:1.3 |
| Core material | Ferrite (N87, N97, or equivalent) or nanocrystalline | Low core loss at 100+ kHz |
| Core geometry | EE, ETD, or toroidal | ETD59 or custom for 30 kW |
| Winding | Primary: Litz wire; Secondary: Litz wire or copper foil | Minimize skin/proximity effect |
| Insulation | Triple-insulated wire (TIW) or bobbin with 8 mm creepage | 4 kV isolation requirement |
| Temperature rise | <50°C above ambient at rated load | With thermal management |
| Leakage inductance | Controlled to form L_r (or supplemented with external inductor) | Critical for LLC resonance |
| Magnetizing inductance | Controlled to form L_m | Critical for ZVS range |
| Weight | ~3–5 kg | Significant portion of module weight |
| Dimensions | ~80×80×60 mm | Core + bobbin + terminals |

#### Output Rectifier

| Parameter | Value | Notes |
|-----------|-------|-------|
| Type | SiC Schottky diodes (preferred) or synchronous rectification (SiC MOSFETs) |
| Quantity | 2 (center-tapped) or 4 (full-bridge) | Depends on secondary topology |
| Voltage rating | 1200V | Full output range + margin |
| Current rating | 80A (per diode, for center-tapped) | Handles full module current |
| Forward voltage (SiC Schottky) | 1.2–1.5V | Lower than Si diode (significant efficiency gain) |
| Reverse recovery | ~0 (Schottky — no reverse recovery loss) | Major advantage of SiC |
| Package | TO-247-3 or module | Mounted on heatsink |

### 5.4 LLC Control

```
                    ┌──────────────────────────────────────────┐
                    │          LLC CONTROL                      │
                    │                                          │
  V_out_ref ──►(+)─┼──►┌──────────┐                           │
  (from CAN)   (-) │   │ Voltage  │──► VCO ──► Gate Driver    │
               │   │   │ PI Loop  │   (Voltage                │
  V_out_meas ──┘   │   │          │    Controlled              │
                    │   └──────────┘    Oscillator)             │
                    │                                          │
                    │   f_sw output:                            │
                    │   - V_out < target → decrease f_sw        │
                    │     (below resonance = voltage boost)     │
                    │   - V_out > target → increase f_sw        │
                    │     (above resonance = voltage buck)       │
                    │                                          │
                    │   Dead-time control:                      │
                    │   - Adaptive dead time ensures ZVS        │
                    │   - Monitored via drain-source voltage    │
                    │                                          │
                    └──────────────────────────────────────────┘

Voltage loop bandwidth: ~2 kHz
Minimum frequency: 80 kHz (maximum voltage boost)
Maximum frequency: 300 kHz (minimum voltage, light load)
Dead time: 100–300 ns (adaptive)
```

### 5.5 Output Filter

| Component | Value | Notes |
|-----------|-------|-------|
| Output capacitance (C_out) | 100–200 µF | Film capacitor, 1200V rated |
| Output inductor (optional) | 10–50 µH | Only if ripple requirements are tight |
| Voltage ripple | <1% of V_out | Achieved by C_out sizing |

## 6. Module Controller

### 6.1 DSP / MCU

The module controller runs the real-time control loops for both PFC and LLC stages, manages protection, and communicates with the Phytec SBC over CAN.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Processor | Texas Instruments TMS320F280049C or equivalent | C2000 real-time DSP |
| Clock speed | 100–200 MHz | Sufficient for dual-loop control at 65–300 kHz |
| PWM channels | 12 (6 for PFC + 4 for LLC + 2 spare) | High-resolution PWM (150 ps) |
| ADC channels | 16 (12-bit, 3.45 MSPS) | Simultaneous sampling for current/voltage |
| CAN interface | 1× CAN 2.0A (to CAN #1 bus) | Isolated via transceiver |
| GPIO | 8+ (for ENABLE input, fault signals, LED, fan PWM) | |
| Flash | 256 KB | Firmware + calibration data |
| RAM | 100 KB | Control loop variables, buffers |
| Packages | LQFP-100 or QFP-64 | |

Alternative MCUs: STM32G474 (for lower cost), Infineon XMC4800 (integrated gate driver peripherals), Microchip dsPIC33CK.

### 6.2 Gate Drivers

Each SiC MOSFET requires an isolated gate driver capable of the +18V / -5V drive levels and fast rise/fall times.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Quantity | 10 total (6 for PFC + 4 for LLC) | One per MOSFET |
| Type | Isolated gate driver IC | e.g., Silicon Labs Si8271, Broadcom ACPL-302J, or Infineon 1EDC series |
| Output voltage | +18V / -5V (with bipolar supply) | SiC requires negative gate for reliable off-state |
| Peak source current | >4A | Fast turn-on |
| Peak sink current | >8A | Fast turn-off (critical for SiC) |
| Isolation voltage | ≥3 kV (reinforced) | Input-to-output |
| Propagation delay | <100 ns | Matched across all channels |
| CMTI (Common Mode Transient Immunity) | >100 kV/µs | Essential for high dv/dt SiC switching |
| Desaturation detection | Yes | Overcurrent protection via gate driver |
| UVLO (Under-Voltage Lock-Out) | Yes | Prevents partial gate drive |
| Dead time insertion | Hardware configurable | 100–300 ns adjustable |

### 6.3 Sensing Circuits

| Measurement | Sensor Type | Range | Resolution | Update Rate |
|-------------|-------------|-------|------------|-------------|
| AC input voltage (per phase) | Resistive divider + diff amp | 0–700V AC | 12-bit | 65 kHz (per PWM cycle) |
| AC input current (per phase) | Rogowski coil or Hall effect | 0–60A RMS | 12-bit | 65 kHz |
| DC link voltage | Resistive divider + isolation amp | 0–1000V | 12-bit | 65 kHz |
| DC output voltage | Isolated resistive divider | 0–1200V | 12-bit | 300 kHz (LLC rate) |
| DC output current | Hall effect (closed-loop) | 0–80A | 12-bit | 300 kHz |
| MOSFET temperature (×2 zones) | NTC 10kΩ on heatsink | -40 to +175°C | 10-bit | 1 kHz |
| Transformer temperature | NTC embedded in winding | -40 to +180°C | 10-bit | 1 kHz |
| Inductor temperature (×3) | NTC on core | -40 to +150°C | 10-bit | 1 kHz |
| DC link cap temperature | NTC on case | -40 to +105°C | 10-bit | 1 kHz |
| PCB temperature | NTC on controller board | -40 to +100°C | 10-bit | 1 Hz |

### 6.4 Protection Circuits

| Protection | Detection | Response Time | Action |
|------------|-----------|---------------|--------|
| Output overvoltage (OVP) | Hardware comparator on V_out | <1 µs | Immediate gate inhibit |
| Output overcurrent (OCP) | Desaturation detection in gate driver | <1 µs | Immediate gate inhibit |
| DC link overvoltage | Hardware comparator | <1 µs | Gate inhibit + crowbar |
| Short circuit | Gate driver desat + high dI/dt detection | <500 ns | Soft turn-off (STO) |
| Over-temperature | NTC comparator (hardware) | <10 ms | Disable PWM, report fault |
| AC input loss | Voltage monitor, phase-loss detection | <20 ms | Controlled shutdown |
| ENABLE input de-asserted | Hardware interrupt | <1 ms | Immediate gate inhibit |
| Watchdog timeout (internal) | DSP watchdog timer | <10 ms | Reset and re-initialize |

**Soft Turn-Off (STO):** For SiC MOSFETs, an abrupt gate turn-off during a short circuit causes dangerous voltage spikes from parasitic inductance (L × dI/dt). The gate driver implements a controlled slow turn-off (~1 µs ramp) to limit the overshoot to safe levels. This is a critical safety feature specific to SiC devices.

### 6.5 ENABLE Hardware Input

The safety supervisor provides a hardware ENABLE signal (active high, 24V logic via optocoupler). If ENABLE is not asserted, the module controller must inhibit all gate drive signals within 1 ms, regardless of CAN commands. This provides a safety-rated shutdown path that is independent of both the CAN bus and the module firmware.

```
From Safety Supervisor DO5
│
├─── 24V signal
│
└─── Optocoupler (input side)
     │
     └─── Optocoupler (output side)
          │
          ├─── To DSP GPIO (ENABLE status input)
          │
          └─── To hardware gate inhibit (AND gate)
               │
               └─── All gate driver enable pins
                    (if ENABLE = LOW, all gates = OFF)
```

## 7. Auxiliary Power Supply

The module requires multiple low-voltage rails for its controller, gate drivers, sensors, and fan.

```
24V DC Input (from PDU 1 or backplane)
│
├─── Isolated DC-DC: 24V → 15V (for gate driver bias supply)
│    Output: 15V, 2A (30W)
│    Isolation: 4 kV
│    Topology: Flyback
│    │
│    └─── Per-gate-driver bipolar supply:
│         15V → +18V / -5V (charge pump or LDO per channel)
│
├─── Buck converter: 24V → 5V
│    Output: 5V, 2A (10W)
│    Purpose: Sensors, analog circuits, fan control
│
├─── Buck converter: 24V → 3.3V
│    Output: 3.3V, 1A (3.3W)
│    Purpose: DSP core and I/O, CAN transceiver
│
└─── Fan output: 24V PWM
     Purpose: Module-local cooling fan
     Max current: 1A
```

Total auxiliary power consumption: ~45W (from 24V input).

## 8. Thermal Design

### 8.1 Heat Dissipation

At 30 kW output and 96% efficiency, the module dissipates ~1.2 kW of heat. This heat is concentrated in the switching devices, magnetic components, and rectifiers.

| Component | Heat Source | Dissipation (W) | Cooling Method |
|-----------|-----------|-----------------|----------------|
| PFC SiC MOSFETs (×6) | Switching + conduction | 360 | Coldplate (liquid) |
| LLC SiC MOSFETs (×4) | Switching + conduction | 276 | Coldplate (liquid) |
| Output rectifier diodes (×2–4) | Forward voltage drop | 120 | Coldplate (liquid) |
| Boost inductors (×3) | Core + copper loss | 180 | Thermal pad to baseplate |
| HF Transformer | Core + copper loss | 180 | Thermal pad to baseplate |
| DC link capacitors | ESR × I²_ripple | 36 | Ambient air + conduction |
| Controller / gate drivers | Logic dissipation | 48 | PCB copper pour + air |
| **Total** | | **~1200W** | |

### 8.2 Coldplate Design

All major power semiconductors and magnetic components are mounted on an aluminum coldplate through which liquid coolant flows.

```
MODULE BASEPLATE (Bottom View)

┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  COOLANT IN ●───────────────────────────────────────● COOLANT │
│             │                                       │ OUT     │
│             │   ┌──────────────────────────────┐    │         │
│             │   │  Serpentine coolant channel   │    │         │
│             └──►│  machined in aluminum        │────┘         │
│                 │  baseplate                    │              │
│                 └──────────────────────────────┘              │
│                                                               │
│  THERMAL INTERFACE MATERIAL (TIM)                             │
│  ═══════════════════════════════════════════════════════════  │
│                                                               │
│  ┌────────┐ ┌────────┐ ┌────────┐  PFC MOSFETs (×6)          │
│  │ Q1a/1b │ │ Q2a/2b │ │ Q3a/3b │  on insulated substrate    │
│  └────────┘ └────────┘ └────────┘                             │
│                                                               │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  LLC MOSFETs    │
│  │   Q3   │ │   Q4   │ │   Q5   │ │   Q6   │  + Rectifiers   │
│  └────────┘ └────────┘ └────────┘ └────────┘                 │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  Boost Inductors (LA, LB, LC)                        │     │
│  │  + HF Transformer                                    │     │
│  │  (thermal pads to baseplate)                         │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                               │
└───────────────────────────────────────────────────────────────┘

Baseplate material: Aluminum 6063-T5
Baseplate thickness: 10–15 mm
Channel depth: 3–5 mm
Channel width: 8–12 mm
Surface finish: Machined, anodized
TIM: Thermal grease (2–5 W/m·K) or thermal pad (3 W/m·K)
```

### 8.3 Coolant Requirements (Per Module)

| Parameter | Value | Notes |
|-----------|-------|-------|
| Coolant type | 50/50 ethylene glycol-water | Same as system coolant |
| Flow rate | 3–5 L/min | Per module |
| Inlet temperature | 30–45°C | From HVAC radiator or ambient |
| Outlet temperature | 40–55°C | ΔT ≈ 5–15°C depending on load |
| Pressure drop | 0.2–0.5 bar | At 4 L/min |
| Maximum coolant temperature | 60°C | Above this → module derates |
| Fitting type | Push-in or barb, 10 mm ID | Quick-connect for hot-swap |

### 8.4 Thermal Derating

| Condition | Derating Action |
|-----------|-----------------|
| MOSFET temp 125–150°C | Reduce output power linearly (100% → 50%) |
| MOSFET temp >150°C (derating start per System Architecture) | Hard derate to 50% |
| MOSFET temp >165°C | Shutdown, fault 0x07 |
| Transformer temp >120°C | Derate to 75% |
| Transformer temp >135°C | Shutdown, fault 0x08 |
| DC link cap temp >85°C | Derate to 75% |
| DC link cap temp >100°C | Shutdown, fault 0x09 |
| Coolant flow loss | Immediate shutdown |
| Coolant temp >60°C | Derate to 50% |

## 9. Mechanical Design

### 9.1 Module Enclosure

| Parameter | Value | Notes |
|-----------|-------|-------|
| Dimensions (H×W×D) | 450×400×250 mm | Per 01 - Hardware Components |
| Weight | ~35 kg | Including coldplate and magnetics |
| Material | Sheet aluminum (top cover) + machined aluminum (baseplate) | |
| IP rating | IP20 (installed in sealed cabinet) | Not independently sealed |
| Mounting | 4× M8 bolt-down to power shelf | With alignment pins |
| Cooling connections | 2× push-in fittings (10 mm, bottom) | Quick-connect for hot-swap |
| AC input | 3-pin terminal block (L1, L2, L3) + PE stud | On rear face |
| DC output | 2-pin terminal block (DC+, DC-) | On rear face |
| CAN interface | 2× RJ45 or 4-pin Molex (daisy-chain in/out) | On front face |
| ENABLE input | 2-pin connector (24V logic) | On front face |
| Status LED | 1× RGB LED (visible from front) | Green=OK, Yellow=Derate, Red=Fault |
| Fan | 1× 80mm axial fan (24V, internal) | For controller and capacitor cooling |

### 9.2 Module Front/Rear Panel

```
FRONT FACE (Service Access Side)

    ┌─────────────────────────────────────────────┐
    │                                             │
    │   ● Status LED (RGB)                        │
    │                                             │
    │   ┌──────┐  ┌──────┐  CAN Daisy-Chain       │
    │   │CAN IN│  │CAN OUT│  (RJ45 or Molex)     │
    │   └──────┘  └──────┘                        │
    │                                             │
    │   ┌────────┐  ENABLE Input                   │
    │   │ ENABLE │  (2-pin, from Safety Sup.)     │
    │   └────────┘                                │
    │                                             │
    │   ┌──────┐  Address DIP Switch               │
    │   │ DIP  │  (Node ID: 0x01–0x0C)            │
    │   └──────┘                                  │
    │                                             │
    │   ▦▦▦▦▦  Fan Grille (80mm)                  │
    │                                             │
    └─────────────────────────────────────────────┘


REAR FACE (Power Connections)

    ┌─────────────────────────────────────────────┐
    │                                             │
    │   ┌────┬────┬────┬────┐  AC Input Terminal   │
    │   │ L1 │ L2 │ L3 │ PE │  (50A rated)        │
    │   └────┴────┴────┴────┘                     │
    │                                             │
    │   ┌─────────┬─────────┐  DC Output Terminal  │
    │   │   DC+   │   DC-   │  (70A rated)        │
    │   └─────────┴─────────┘                     │
    │                                             │
    │   ● Coolant IN    ● Coolant OUT              │
    │   (10mm push-in)  (10mm push-in)            │
    │                                             │
    └─────────────────────────────────────────────┘
```

### 9.3 Power Shelf Mounting

Modules slide into a power shelf (rack) inside the cabinet. Each shelf holds 4 modules. The shelf provides:
- Mechanical guide rails for module insertion
- Backplane AC power bus connections (auto-mating spring contacts or bolt-down)
- DC output bus connections (bolt-down bus bars)
- Coolant manifold with quick-disconnect per module
- CAN daisy-chain wiring (pre-routed in shelf)

```
POWER SHELF (Top View, 4 Modules)

    ┌────────────────────────────────────────────────────────────────────┐
    │                        POWER SHELF                                │
    │                                                                    │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
    │  │Module 1  │  │Module 2  │  │Module 3  │  │Module 4  │          │
    │  │  30 kW   │  │  30 kW   │  │  30 kW   │  │  30 kW   │          │
    │  │          │  │          │  │          │  │          │          │
    │  │   FRONT  │  │   FRONT  │  │   FRONT  │  │   FRONT  │          │
    │  │  (CAN,   │  │  (CAN,   │  │  (CAN,   │  │  (CAN,   │  ← Service
    │  │  ENABLE, │  │  ENABLE, │  │  ENABLE, │  │  ENABLE, │    access
    │  │   LED)   │  │   LED)   │  │   LED)   │  │   LED)   │          │
    │  └──────────┘  └──────────┘  └──────────┘  └──────────┘          │
    │       │  │          │  │          │  │          │  │              │
    │  ═════╪══╪══════════╪══╪══════════╪══╪══════════╪══╪═════════    │
    │       │  │   AC BUS (L1, L2, L3, PE per module)  │  │             │
    │  ═════╪══╪══════════╪══╪══════════╪══╪══════════╪══╪═════════    │
    │       │  │   DC BUS (DC+, DC- shared)  │  │      │  │             │
    │  ─────┼──┼──────────┼──┼──────────┼──┼──────────┼──┼─────────    │
    │       ●  ●          ●  ●          ●  ●          ●  ●              │
    │     Coolant       Coolant       Coolant       Coolant             │
    │     IN/OUT        IN/OUT        IN/OUT        IN/OUT              │
    │                                                                    │
    │  ════════════════ COOLANT MANIFOLD ═══════════════════════════    │
    │                                                                    │
    └────────────────────────────────────────────────────────────────────┘
```

## 10. EMC Design

### 10.1 EMI Filter (Input)

Each module has an input EMI filter to meet conducted emission limits per IEC 61000-6-4 (industrial) and CISPR 11 Class A.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Filter topology | Two-stage LC (common-mode + differential-mode) | |
| CM choke | 2× common-mode toroid, 2–5 mH | Nanocrystalline core |
| DM capacitors | 3× 2.2 µF, X2 rated (line-to-line) | Film capacitors |
| CM capacitors | 6× 4.7 nF, Y2 rated (line-to-ground) | Ceramic, safety rated |
| DM inductors | 3× 10–50 µH (integrated in CM choke or separate) | |
| Attenuation | >60 dB at switching frequency (65 kHz) | |
| Leakage current | <3.5 mA per module (at 50 Hz) | IEC 62477-1 limit |

### 10.2 PCB Layout Considerations

- **Power loop minimization**: Gate driver located within 10 mm of MOSFET; decoupling capacitor between source and gate driver ground
- **Kelvin source connections**: Four-lead SiC MOSFETs use separate source pins for gate drive and power to avoid common-source inductance
- **Shielding**: Controller PCB shielded from power stage by copper pour and physical separation
- **Ground planes**: Separate analog ground, digital ground, and power ground planes; single-point star connection at DSP
- **Creepage/clearance**: 8 mm minimum creepage on PCB for 1000V DC circuits

## 11. Module Self-Test and Diagnostics

### 11.1 Power-On Self-Test (POST)

| Test | Method | Pass Criteria | Duration |
|------|--------|---------------|----------|
| DSP core | RAM pattern test, flash CRC | Match | 50 ms |
| ADC calibration | Read internal reference | ±2% of expected | 10 ms |
| Gate driver check | Toggle each driver, verify UVLO and desat | All drivers report ready | 20 ms |
| Temperature sensors | Read all NTCs | All in -40 to +85°C range | 5 ms |
| CAN loopback | Send/receive test frame | Frame received | 10 ms |
| Fan test | Run fan at 50%, check tachometer | RPM within expected range | 500 ms |
| Isolation test (optional) | Low-voltage insulation check | >1 MΩ | 100 ms |
| **Total POST time** | | | **~700 ms** |

### 11.2 Runtime Diagnostics

| Diagnostic | Period | Action on Failure |
|------------|--------|-------------------|
| ADC reference drift | 1 s | Fault 0x0E (self-test fail) |
| Gate driver fault (desat) | Per switching cycle | Fault 0x0B (short circuit) |
| Phase current imbalance | 100 ms | Warning 0x0F, then fault if >20% |
| DC link voltage range | Every PFC cycle | Fault 0x03 (OV) or 0x04 (UV) |
| Efficiency monitor | 1 s | Warning if <90% (possible component degradation) |
| Fan tachometer | 1 s | Warning 0x0A (fan failure) |
| CAN heartbeat (master) | 1 s | If lost >2 s → ramp to zero, enter standby |

## 12. Bill of Materials (Per Module)

### 12.1 Power Semiconductors

| Item | Qty | Specification | Reference |
|------|-----|---------------|-----------|
| SiC MOSFET 1200V/60A (PFC) | 6 | TO-247-4, 25–40 mΩ | Wolfspeed C3M0025120K or equiv. |
| SiC MOSFET 1200V/40A (LLC) | 4 | TO-247-4, 40–60 mΩ | Wolfspeed C3M0040120K or equiv. |
| SiC Schottky diode 1200V/80A (rectifier) | 4 | TO-247-3 | Wolfspeed C4D80120D or equiv. |
| Isolated gate driver IC | 10 | ≥4A source, ≥8A sink, desat detect | Si8271, ACPL-302J, or equiv. |

### 12.2 Magnetic Components

| Item | Qty | Specification |
|------|-----|---------------|
| PFC boost inductor | 3 | 200–400 µH, 50A, nanocrystalline core, Litz wire |
| LLC resonant inductor | 1 | 10–30 µH (may be integrated in transformer) |
| HF transformer | 1 | 30 kW, 80–300 kHz, ferrite/nanocrystalline, 4 kV isolation |
| CM choke (EMI filter) | 2 | 2–5 mH, nanocrystalline toroid |

### 12.3 Capacitors

| Item | Qty | Specification |
|------|-----|---------------|
| DC link film capacitors (upper bank) | 4–6 | 100–250 µF, 500V, polypropylene, low ESR |
| DC link film capacitors (lower bank) | 4–6 | 100–250 µF, 500V, polypropylene, low ESR |
| LLC resonant capacitor | 1–2 | 30–100 nF, 1200V, film, high ripple current |
| Output filter capacitor | 2–4 | 47–100 µF, 1200V, film |
| EMI filter X2 capacitors | 3 | 2.2 µF, 310V AC, film |
| EMI filter Y2 capacitors | 6 | 4.7 nF, 250V AC, ceramic safety rated |

### 12.4 Control and Sensing

| Item | Qty | Specification |
|------|-----|---------------|
| DSP/MCU | 1 | TMS320F280049C or equiv., LQFP-100 |
| CAN transceiver (isolated) | 1 | ISO 11898-2, 3 kV isolation |
| Current sensor (AC phases) | 3 | Rogowski coil or Hall, 60A range |
| Current sensor (DC output) | 1 | Closed-loop Hall, 80A range |
| NTC 10kΩ temperature sensors | 6 | For MOSFETs, transformer, inductors, caps, PCB |
| Voltage dividers (isolated) | 3 | AC input, DC link, DC output |
| Optocoupler (ENABLE input) | 1 | 3 kV isolation, fast response |

### 12.5 Mechanical and Thermal

| Item | Qty | Specification |
|------|-----|---------------|
| Aluminum coldplate (machined) | 1 | 400×250×15 mm, with serpentine channel |
| Thermal interface material (TIM) | 1 set | Pads or grease, 3–5 W/m·K |
| Module enclosure (top cover) | 1 | Sheet aluminum, powder-coated |
| Axial fan (80mm) | 1 | 24V DC, 50 CFM, PWM, tachometer |
| Coolant fittings (10mm push-in) | 2 | Quick-disconnect |
| Input fuse holder + fuses | 1 set | 3-pole, 63A gG |
| AC terminal block | 1 | 4-position (L1, L2, L3, PE), 63A rated |
| DC terminal block | 1 | 2-position (DC+, DC-), 70A rated |
| CAN connectors | 2 | RJ45 or 4-pin Molex |
| ENABLE connector | 1 | 2-pin Molex or JST |
| Address DIP switch | 1 | 4-position |
| Status LED (RGB) | 1 | Panel-mount |
| M8 mounting bolts + alignment pins | 4+2 | For shelf mounting |

## 13. Module Firmware Overview

The module DSP firmware is structured for real-time control with hard timing guarantees:

```
┌──────────────────────────────────────────────────────────────────┐
│                    MODULE FIRMWARE LAYERS                         │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  INTERRUPT SERVICE ROUTINES (Highest Priority)             │  │
│  │                                                            │  │
│  │  PWM ISR (65 kHz / PFC):                                  │  │
│  │    - Read AC voltage/current ADCs                         │  │
│  │    - Execute PFC current loop                              │  │
│  │    - Update PFC PWM duty cycles                           │  │
│  │    - Check hardware OVP/OCP comparators                   │  │
│  │                                                            │  │
│  │  PWM ISR (variable / LLC):                                │  │
│  │    - Read DC output voltage/current ADCs                  │  │
│  │    - Execute LLC voltage loop                              │  │
│  │    - Update LLC switching frequency                       │  │
│  │    - Adaptive dead-time calculation                       │  │
│  │                                                            │  │
│  │  Fault ISR:                                                │  │
│  │    - Gate driver desat interrupt                          │  │
│  │    - Hardware comparator trip                              │  │
│  │    - Immediate gate inhibit + fault latch                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  MAIN LOOP (Background, 1 kHz tick)                       │  │
│  │                                                            │  │
│  │  - PFC voltage outer loop (20 Hz update)                  │  │
│  │  - Temperature monitoring and derating calculation        │  │
│  │  - CAN message processing (RX commands, TX status)        │  │
│  │  - State machine (Init → Ready → Standby → CV/CC → Fault)│  │
│  │  - Fan speed control (PID on MOSFET temp)                 │  │
│  │  - Self-diagnostics                                       │  │
│  │  - Watchdog feed                                          │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  HAL / DRIVERS                                             │  │
│  │  PWM, ADC, CAN, GPIO, SPI (for gate drivers), Timer       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

Key timing constraints:
  PFC current loop: 15.4 µs period (65 kHz), must complete in <10 µs
  LLC voltage loop: 3.3–12.5 µs period (80–300 kHz), must complete in <2 µs
  Fault response: <500 ns (hardware) + <1 µs (soft turn-off)
  CAN TX/RX: 100 ms status cycle, 10 ms emergency latency
```

## 14. Testing and Qualification

### 14.1 Module-Level Tests

| Test | Method | Pass Criteria |
|------|--------|---------------|
| Full-power burn-in | 30 kW at 400V/75A, 48 hours continuous | No fault, temp stable within ±5°C |
| Efficiency test | Power analyzer at 10%, 25%, 50%, 75%, 100% load | Meets spec per Section 3.4 |
| Power factor / THD | Power analyzer at rated load | PF >0.99, THD <5% |
| Voltage regulation | Load step 25% → 100% → 25% | ±0.5%, <5% overshoot |
| Current sharing | 5 modules in parallel, measure current spread | <5% imbalance at rated load |
| Over-temperature shutdown | Block coolant flow, monitor response | Shutdown before T_junction >175°C |
| OVP / OCP | Inject fault via external load | Trip within spec, no damage |
| Insulation test | 4 kV DC, 60 s, input to output | No breakdown, leakage <1 mA |
| EMC conducted emissions | CISPR 11, Class A | Pass |
| EMC radiated emissions | CISPR 11, Class A | Pass |
| Vibration | IEC 60068-2-6 (sinusoidal) | No mechanical failure |
| Thermal cycling | -40°C to +85°C, 100 cycles | No delamination, solder joint integrity |

### 14.2 Hot-Swap Test

| Step | Condition | Verification |
|------|-----------|--------------|
| 1 | 5 modules charging at 150 kW (5 × 30 kW) | Stable operation |
| 2 | Disconnect Module 3 (pull from shelf) | Remaining 4 modules absorb load within 200 ms |
| 3 | Insert replacement Module 3 | Module detected via heartbeat, enters standby |
| 4 | Phytec SBC enables replacement module | Current redistributed across 5 modules |
| 5 | Verify no session interruption | EV continues charging at full rate |

## 15. References

- IEC 62477-1: Safety requirements for power electronic converter systems
- IEC 61000-3-12: Harmonic current limits (equipment >16A)
- IEC 61000-6-4: EMC generic emission standard (industrial)
- CISPR 11: Industrial, scientific and medical equipment — Radio disturbance characteristics
- [[01 - Hardware Components]] — Module ratings and BOM categories
- [[02 - Electric Wiring Diagram]] — AC input and DC output connections
- [[03 - Cabinet Layout]] — Physical placement in power electronics zone
- [[04 - Backplane Power Management]] — PDU 1 feeds to power modules
- [[05 - DC Output Contactor and Pre-Charge Circuit]] — Downstream switching
- [[docs/System/01 - System Architecture|01 - System Architecture]] — Modular power architecture, thermal zones
- [[docs/Software/04 - Power Module CAN Bus Interface|04 - Power Module CAN Bus Interface]] — CAN message dictionary
- [[docs/Software/EVerest/02 - EVerest Power Module Driver|02 - EVerest Power Module Driver]] — EVerest software interface
- [[docs/Software/03 - Safety Supervisor Controller|03 - Safety Supervisor Controller]] — ENABLE signal and safety shutdown

---

**Document Version**: 1.1
**Last Updated**: 2026-02-27
**Prepared by**: Power Electronics Engineering
