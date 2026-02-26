# DC Output Contactor and Pre-Charge Circuit

Tags: #dcfc #hardware #contactor #precharge #power-electronics #safety

Related: [[01 - Hardware Components]] | [[02 - Electric Wiring Diagram]] | [[04 - Backplane Power Management]] | [[docs/Software/03 - Safety Supervisor Controller|03 - Safety Supervisor Controller]]

## 1. Overview

The DC output contactor assembly and pre-charge circuit form the critical switching interface between the power conversion modules and the EV charging connector. This assembly must safely connect and disconnect high-voltage DC (200–1000V, up to 500A) under both normal operating conditions and fault scenarios. The pre-charge circuit prevents damaging inrush current into the EV's input filter capacitors by ramping the output voltage in a controlled manner before the main contactors close.

These components sit between the DC link capacitor bank (output of the power conversion stage) and the charging cable/connector. They are controlled exclusively by the Safety Supervisor Controller (STM32), not directly by the CM5 main controller.

## 2. System Context

```
  POWER MODULE                   DC OUTPUT CONTACTOR ASSEMBLY                    TO EV
  DC OUTPUT                                                                     CONNECTOR
                     ┌──────────────────────────────────────────────┐
                     │                                              │
  DC+ ──────────────►│──┬── DC Breaker CB1 ──┬── Fuse F1 ───────────│──────► DC+
                     │  │                    │                      │
                     │  │    ┌───────────┐   │                      │
                     │  │    │ Pre-Charge│   │                      │
                     │  ├────┤ Resistor  ├───┤                      │
                     │  │    │ R1 (100Ω) │   │                      │
                     │  │    └─────┬─────┘   │                      │
                     │  │          │         │                      │
                     │  │    ┌─────┴─────┐   │                      │
                     │  │    │ Pre-Charge│   │                      │
                     │  │    │Contactor  │   │                      │
                     │  │    │   K4      │   │                      │
                     │  │    └───────────┘   │                      │
                     │  │                    │                      │
                     │  │    ┌───────────┐   │                      │
                     │  └────┤ Main DC+  ├───┘                      │
                     │       │Contactor  │                          │
                     │       │   K2      │                          │
                     │       └───────────┘                          │
                     │                                              │
  DC- ──────────────►│──── Fuse F2 ──── Main DC- Contactor K3 ──────│──────► DC-
                     │                                              │
                     │  ┌──────────────┐   ┌──────────────┐         │
                     │  │ Hall Effect  │   │   Voltage    │         │
                     │  │Current Sensor│   │   Sensor     │         │
                     │  │  (0-600A)    │   │  (0-1200V)   │         │
                     │  └──────────────┘   └──────────────┘         │
                     │                                              │
                     │  ┌──────────────┐                            │
                     │  │  Discharge   │                            │
                     │  │  Resistor    │                            │
                     │  │  R2 (10kΩ)   │                            │
                     │  └──────────────┘                            │
                     │                                              │
                     └──────────────────────────────────────────────┘
```

## 3. Main DC Output Contactors (K2, K3)

### 3.1 Purpose

The main DC contactors provide galvanic isolation between the charger's DC bus and the EV connector. They are the primary means of connecting and disconnecting high-voltage DC power to the vehicle. Both the positive rail (K2) and negative rail (K3) are independently switched to achieve full isolation — no live potential remains at the connector when the contactors are open.

### 3.2 Contactor Selection Criteria

| Parameter | Requirement | Rationale |
|-----------|-------------|-----------|
| Rated voltage | ≥1000V DC | Full output range (200–1000V) |
| Rated current | ≥500A continuous | Maximum charging current |
| Making capacity | ≥1000A | Inrush during pre-charge bypass transition |
| Breaking capacity | ≥500A at 1000V DC | Must interrupt full-load current under fault |
| Coil voltage | 24V DC | Matches control supply rail |
| Coil power | 15–30W (pull-in), 3–8W (hold) | Economy coil with PWM hold reduces heat |
| Auxiliary contacts | Min. 1 NO + 1 NC per contactor | Feedback to safety supervisor (DI5, DI6) |
| Mechanical life | ≥500,000 operations | ~10 years at 50,000 sessions/year |
| Electrical life | ≥100,000 operations at rated load | Per IEC 60947-4-1 |
| Contact material | AgSnO₂ or AgCdO | Low contact resistance, arc resistance |
| Arc suppression | Internal magnetic blowout or gas-filled chamber | Reliable DC arc extinguishing |
| Response time | Close: <50 ms, Open: <30 ms | Safety response time budget |

### 3.3 Contactor Specifications

| Parameter | K2 (DC+ Contactor) | K3 (DC- Contactor) |
|-----------|---------------------|---------------------|
| Application | Positive rail switching | Negative rail switching |
| Rating | 500A, 1000V DC | 500A, 1000V DC |
| Coil voltage | 24V DC | 24V DC |
| Coil current (pull-in) | ~1.2A | ~1.2A |
| Coil current (hold) | ~0.3A (PWM) | ~0.3A (PWM) |
| Auxiliary contacts | 2 NO + 2 NC | 1 NO + 1 NC |
| Contact resistance | <0.5 mΩ | <0.5 mΩ |
| Mounting | Bolt-down, M8 (4x) | Bolt-down, M8 (4x) |
| Weight | ~2.5 kg | ~2.5 kg |
| Dimensions (approx.) | 100×80×90 mm | 100×80×90 mm |

### 3.4 Coil Drive Circuit

Each contactor coil is driven by the safety supervisor through an isolated MOSFET driver with flyback protection. The coil uses PWM hold current to reduce steady-state power dissipation.

```
Safety Supervisor DO2 (DC Main Contactor)
│
├─── Optocoupler (isolation)
│    │
│    └─── Gate Driver IC
│         │
│         └─── N-Channel MOSFET (Q1)
│              Drain ──── K2 Coil (+) ────── 24V DC (via safety relay K10)
│              Source ─── GND
│              │
│              └─── Flyback Diode (D2)
│                   Type: Fast recovery, TVS clamped
│                   Across coil terminals
│                   Clamp voltage: 48V (2× coil voltage)
│
│    PWM Control:
│    - Pull-in: 100% duty, 200 ms duration
│    - Hold: 30-40% duty (reduces coil heating from 30W to ~8W)
│    - Off: 0% duty (contactor opens)
│
│    Feedback Path:
│    K2 Auxiliary NC contact ──► Safety Supervisor DI5
│    (NC = closed when contactor is open → confirms open state)
│    (NC = open when contactor is closed → confirms closed state)
```

### 3.5 Contactor Weld Detection

A welded contactor is a critical safety hazard — it means the charger cannot isolate the EV from high voltage. The safety supervisor detects this condition:

```
WELD DETECTION LOGIC (runs after every open command):

1. Command contactor OPEN (DO = 0)
2. Wait 50 ms (mechanical release time)
3. Read auxiliary feedback contact:
   - NC contact should return to CLOSED (= contactor is open) → OK
   - NC contact remains OPEN (= contactor still closed) → WELD DETECTED
4. Additionally, check output voltage:
   - If V_output > 60V after 500 ms with both contactors commanded open
     → possible weld on at least one contactor

On weld detection:
  - Latch fault F05 (CONTACTOR_WELD)
  - Do NOT attempt to re-close any contactor
  - Report to CM5 → OCPP StatusNotification (faulted)
  - Charger is out of service until field repair
```

### 3.6 Why Both Rails Are Switched

Switching only the positive rail would leave the negative rail connected to the EV at all times, creating:
- A single-fault-away-from-shock condition (if positive insulation degrades)
- Inability to fully de-energize the connector for safe cable handling
- Non-compliance with IEC 61851-23 which requires full galvanic isolation

Dual-contactor switching ensures that a single contactor weld still leaves one point of isolation, and the safety supervisor can detect the welded contactor via feedback before the next session begins.

## 4. Pre-Charge Circuit

### 4.1 Purpose

When the main DC contactors close, the charger's DC bus (at 200–1000V) connects to the EV's input filter capacitors and internal DC bus. These capacitors are initially discharged (or at a lower voltage). Without pre-charge, closing the contactors directly would cause:

- **Inrush current surge** of thousands of amps (limited only by cable and contact resistance)
- **Contact arc welding** from the high inrush at the moment of closure
- **Voltage sag** on the charger's DC link, potentially tripping OVP/OCP protections
- **Stress on EV input capacitors** (exceeding ripple current ratings)

The pre-charge circuit limits this inrush by inserting a resistor in series, allowing the output voltage to ramp gradually to match the DC bus voltage before the main contactors close.

### 4.2 Circuit Topology

```
          DC+ from power module
               │
               │
          ┌────┴────┐
          │         │
     ┌────┴────┐   ┌┴─────────────┐
     │         │   │              │
     │  Main   │   │  Pre-Charge  │
     │  Path   │   │  Path        │
     │         │   │              │
     │  ┌───┐  │   │  ┌────┐      │
     │  │K2 │  │   │  │ R1 │      │
     │  │   │  │   │  │100Ω│      │
     │  └─┬─┘  │   │  └──┬─┘      │
     │    │    │   │     │        │
     │    │    │   │  ┌──┴──┐     │
     │    │    │   │  │ K4  │     │
     │    │    │   │  │     │     │
     │    │    │   │  └──┬──┘     │
     │    │    │   │     │        │
     └────┬────┘   └─────┬────────┘
          │              │
          └──────┬───────┘
                 │
            To DC Fuse F1
                 │
            To EV Connector DC+
```

The pre-charge path (R1 + K4) is in parallel with the main contactor K2. During pre-charge:
- K2 is **open** (main path blocked)
- K4 is **closed** (current flows through R1)
- Current is limited by R1 to a safe level

Once the output voltage matches the DC bus, K2 closes (low-stress closure since voltage is equalized), and K4 opens (R1 removed from circuit).

### 4.3 Pre-Charge Resistor (R1) Design

#### 4.3.1 Resistance Value Selection

The resistor value determines the maximum inrush current and the RC time constant for voltage ramp-up.

**Peak inrush current at closure:**

```
I_peak = V_bus / R1

At V_bus = 1000V, R1 = 100Ω:
  I_peak = 1000V / 100Ω = 10A

At V_bus = 400V, R1 = 100Ω:
  I_peak = 400V / 100Ω = 4A
```

This is well within the pre-charge contactor K4's rating and avoids arc welding at closure.

#### 4.3.2 Time Constant

The RC time constant depends on the total capacitance being charged (EV input capacitors + charger output filter):

```
τ = R1 × C_total

Typical EV input capacitance: 500 µF to 2000 µF
Charger output filter: 100 µF

For C_total = 1000 µF, R1 = 100Ω:
  τ = 100 × 0.001 = 0.1 s (100 ms)

Time to reach 95% of V_bus (3τ):
  t_95% = 3 × 0.1 = 0.3 s (300 ms)

Time to reach 99% of V_bus (5τ):
  t_99% = 5 × 0.1 = 0.5 s (500 ms)
```

The safety supervisor allows up to **3 seconds** for pre-charge completion before declaring a timeout fault (F06). This accommodates EVs with larger input capacitance (up to ~6000 µF).

#### 4.3.3 Energy Dissipation

During pre-charge, the resistor absorbs energy equal to the energy stored in the capacitor:

```
E_resistor = ½ × C_total × V_bus²

At V_bus = 1000V, C_total = 2000 µF:
  E_resistor = ½ × 0.002 × 1000² = 1000 J (1 kJ)

At V_bus = 400V, C_total = 2000 µF:
  E_resistor = ½ × 0.002 × 400² = 160 J
```

#### 4.3.4 Resistor Specification

| Parameter | Value | Notes |
|-----------|-------|-------|
| Resistance | 100Ω ±5% | Wirewound or thick-film |
| Continuous power | 100W | For repeated pre-charge cycles |
| Pulse energy rating | ≥1500 J | Covers worst-case 1000V into 2000 µF with margin |
| Voltage rating | ≥1200V | Must withstand full DC bus voltage |
| Temperature coefficient | <200 ppm/°C | Resistance stability over temperature |
| Construction | Ceramic-core wirewound or TO-247 thick-film | High pulse capability |
| Mounting | Chassis mount with thermal pad | Heat sinks to cabinet metalwork |
| Dimensions (approx.) | 150×30×30 mm | Wirewound type |

### 4.4 Pre-Charge Contactor (K4)

The pre-charge contactor has significantly lower current requirements than the main contactors since it only carries the limited pre-charge current.

| Parameter | K4 Specification |
|-----------|-----------------|
| Rated voltage | ≥1000V DC |
| Rated current | 50A continuous |
| Making capacity | 50A (no inrush through K4 — R1 limits current) |
| Breaking capacity | 10A at 1000V DC (opens after main contactor has closed) |
| Coil voltage | 24V DC |
| Coil current | ~0.5A |
| Auxiliary contacts | 1 NO (feedback to safety supervisor DI6) |
| Mechanical life | ≥500,000 operations |
| Response time | Close: <30 ms, Open: <20 ms |
| Mounting | DIN rail or bolt-down |
| Weight | ~0.5 kg |

K4 is smaller and less expensive than K2/K3 because:
- Maximum current through R1 is only 10A (at 1000V)
- It never breaks full load current (K2 closes first, then K4 opens with near-zero current)
- No arc suppression challenge since the current at opening is negligible

### 4.5 Discharge Resistor (R2)

A bleed-down resistor across the DC output terminals ensures that residual voltage is safely discharged when the contactors open and the EV disconnects.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Resistance | 10 kΩ | Limits continuous power dissipation |
| Power rating | 100W | At 1000V: P = V²/R = 100W |
| Voltage rating | ≥1200V | |
| Discharge time (1000V → 60V) | ~28 s | 2.8τ, where τ = R2 × C_output |
| Discharge time (1000V → <1V) | ~70 s | Per IEC 61851-23 (< 120 s to safe voltage) |
| Purpose | Safety discharge to <60V DC | IEC 61851-23 Clause 6.4.5 |

```
Discharge time calculation:
  V(t) = V₀ × e^(-t/RC)

  For V₀ = 1000V, R2 = 10kΩ, C = 1000µF:
    τ = 10000 × 0.001 = 10 s
    Time to 60V: t = -τ × ln(60/1000) = 10 × 2.81 = 28.1 s
    Time to 1V:  t = -τ × ln(1/1000)  = 10 × 6.91 = 69.1 s
```

The discharge resistor is permanently connected across DC+ and DC- output terminals (after the main contactors, before the connector). It is always active whenever voltage is present at the output, including when the EV is connected.

## 5. Pre-Charge Operating Sequence

### 5.1 Normal Pre-Charge Sequence

```
Time ──────────────────────────────────────────────────────────────────────────►

PHASE 1: ISOLATION VERIFICATION
│
├── Safety supervisor verifies all inputs OK
├── IMD confirms insulation resistance OK
├── Connector lock confirmed (DI7)
│
▼
PHASE 2: AC CONTACTOR CLOSE (handled by PDU 1)
│
├── K-PM1 / K-PM2 close → power modules energized
├── DC link capacitors charged internally by power modules
├── DC bus voltage present (700-800V on DC link)
│
▼
PHASE 3: PRE-CHARGE
│                                              DC Output Voltage
│   t=0: K4 closes (pre-charge contactor)          │
│         K2 remains open                           │  ┌─────── V_bus
│         K3 closes (negative rail)                 │  │
│                                                   │ /
│   Current flows: DC+ → R1 → K4 → cable → EV →    │/
│                  cable → K3 → DC-                 /
│                                                  /│
│   t=100ms: V_out reaches ~63% of V_bus         / │
│   t=300ms: V_out reaches ~95% of V_bus       /   │
│   t=500ms: V_out reaches ~99% of V_bus     ─     │
│                                                   └─── 0V
│   Safety supervisor monitors:                 0    t(ms)   500
│   - V_out ramp rate (must be rising)
│   - V_out approaching V_bus (within ±5%)
│   - Timeout (3 s max, else fault F06)
│
▼
PHASE 4: MAIN CONTACTOR CLOSURE
│
│   Once |V_out - V_bus| < 5% of V_bus:
│
│   t+0ms:   K2 closes (main DC+ contactor)
│            Minimal inrush since V is equalized
│   t+50ms:  K2 auxiliary feedback verified (closed)
│   t+100ms: K4 opens (pre-charge contactor)
│            No arc since K2 now carries all current
│   t+150ms: K4 auxiliary feedback verified (open)
│
▼
PHASE 5: CHARGING READY
│
├── Safety supervisor asserts ENABLE to power modules
├── Power modules begin current ramp per CCS protocol
├── State: CHARGING
```

### 5.2 Pre-Charge Failure Modes

| Failure | Symptom | Detection | Response |
|---------|---------|-----------|----------|
| R1 open circuit | V_out stays at 0V | No voltage ramp after K4 close | F06 (precharge timeout) |
| R1 short circuit | V_out jumps instantly, high inrush | dV/dt too fast (>10V/ms) | Immediate open K4, F06 |
| K4 weld | R1 stays in circuit after open command | K4 feedback shows closed after open | F05 (contactor weld) |
| K4 fails to close | V_out stays at 0V | No feedback + no voltage ramp | F10 (feedback mismatch) |
| EV capacitance too large | Slow ramp, may timeout | V_out not reaching target in 3 s | F06 (precharge timeout) |
| EV internal short | V_out stays low, high current | Voltage fails to rise despite current flow | F06 + possible F02 (OCP) |
| DC bus voltage absent | Nothing happens | V_bus < 200V at start of precharge | Abort, check power modules |

### 5.3 Pre-Charge dV/dt Monitoring

The safety supervisor monitors the rate of voltage rise during pre-charge to detect abnormal conditions:

```
Normal dV/dt (first 100 ms):
  ΔV/Δt ≈ V_bus / (R1 × C_total)

  At 800V, 100Ω, 1000µF:
    dV/dt ≈ 800 / (100 × 0.001) = 8000 V/s = 8 V/ms (initial)

Abnormal conditions:
  dV/dt > 50 V/ms  → R1 short or very low EV capacitance → WARNING
  dV/dt < 0.5 V/ms → R1 degraded or EV drawing excessive current → WARNING
  dV/dt = 0         → R1 open or K4 not closed → FAULT after 500 ms
```

## 6. DC Protection Devices

### 6.1 DC Circuit Breaker (CB1)

Located upstream of the contactor assembly, CB1 provides backup overcurrent protection that trips independently of the safety supervisor.

| Parameter | Value |
|-----------|-------|
| Type | Molded case, DC rated |
| Rated voltage | 1000V DC |
| Rated current | 500A |
| Breaking capacity | 10 kA at 1000V DC |
| Trip characteristic | Thermal-magnetic |
| Auxiliary contacts | 1 NO + 1 NC (to safety supervisor DI4) |
| Mounting | Bolt-down |

CB1 is the last line of defense — it trips only if the safety supervisor fails to open the contactors during an overcurrent event. Its trip curve is coordinated to be slower than the safety supervisor's electronic OCP response but faster than the cable's thermal damage threshold.

### 6.2 DC Semiconductor Fuses (F1, F2)

| Parameter | Value |
|-----------|-------|
| Type | gR semiconductor fuse (ultra-fast) |
| Rated voltage | 1200V DC |
| Rated current | 500A |
| Breaking capacity | 50 kA |
| I²t (melting) | Coordinated with contactor and cable ratings |
| Mounting | Bolt-in fuse holder |
| Purpose | Protect against catastrophic short circuit beyond CB1's capacity |

F1 and F2 are placed on the DC+ and DC- rails respectively. They protect the wiring and contactors from short-circuit currents exceeding CB1's breaking capacity, particularly in case of a cable-to-ground or cable-to-cable fault at the connector end.

### 6.3 Protection Coordination

```
FAULT CURRENT vs. RESPONSE TIME

Current ──────────────────────────────────────────────────►
         500A        1000A        5000A        50000A

         ┌────────────────────────────────────────────┐
         │  Safety Supervisor Electronic OCP           │
         │  Response: < 10 ms at >550A                 │
         │  Action: Open contactors K2, K3              │
         └────────────────────────────────────────────┘

                     ┌───────────────────────────────┐
                     │  DC Breaker CB1                │
                     │  Magnetic trip: < 50 ms at 5kA │
                     │  Thermal trip: 10s at 110%      │
                     └───────────────────────────────┘

                                        ┌──────────┐
                                        │ Fuse F1  │
                                        │ < 5 ms   │
                                        │ at 10kA+ │
                                        └──────────┘

The hierarchy ensures:
  1. Safety supervisor trips first for moderate overcurrent (software)
  2. CB1 trips for sustained overcurrent if supervisor fails (electromechanical)
  3. Fuses blow for bolted short circuits (fastest, non-resettable)
```

## 7. Measurement and Sensing

### 7.1 DC Output Voltage Sensor

| Parameter | Value |
|-----------|-------|
| Type | Isolated resistive divider + differential amplifier |
| Measurement range | 0–1200V DC |
| Output signal | 0–10V analog (to safety supervisor AI0) |
| Isolation | ≥4 kV |
| Accuracy | ±0.5% of full scale |
| Bandwidth | DC to 10 kHz |
| Response time | <100 µs |
| Location | After main contactors, before fuses (measures output voltage) |

A second voltage measurement point on the charger's DC link (before the contactors) allows the safety supervisor to compare V_bus with V_output. The delta between them drives the pre-charge completion decision.

### 7.2 DC Output Current Sensor

| Parameter | Value |
|-----------|-------|
| Type | Closed-loop Hall effect |
| Measurement range | ±600A |
| Output signal | 4–20 mA or ±10V analog (to safety supervisor AI1) |
| Accuracy | ±0.5% |
| Bandwidth | DC to 100 kHz |
| Response time | <1 µs |
| Power supply | ±15V or 24V DC |
| Location | DC+ rail, between contactor K2 and fuse F1 |

The current sensor serves dual purposes:
- **During pre-charge**: Monitors inrush current to detect resistor or capacitor faults
- **During charging**: Provides independent OCP monitoring by the safety supervisor (separate from the power module's internal current measurement)

## 8. Physical Layout

### 8.1 Component Placement Within Cabinet

```
┌────────────────────────────────────────────────────────────────────────┐
│                     DC OUTPUT CONTACTOR ASSEMBLY                       │
│                     (Located between Power Module zone and             │
│                      Cable Management zone)                           │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                                                                  │  │
│  │  ┌──────────┐                                                    │  │
│  │  │   CB1    │ DC Circuit Breaker                                 │  │
│  │  │  500A    │ (Bolt-down, rear panel)                            │  │
│  │  └────┬─────┘                                                    │  │
│  │       │                                                          │  │
│  │  ┌────┴──────────────────────────────────────────────────────┐   │  │
│  │  │                                                            │   │  │
│  │  │  ┌──────────┐    ┌──────────┐    ┌──────────┐              │   │  │
│  │  │  │ Contactor│    │Pre-Charge│    │Pre-Charge│              │   │  │
│  │  │  │    K2    │    │ Resistor │    │Contactor │              │   │  │
│  │  │  │  DC+ Main│    │    R1    │    │    K4    │              │   │  │
│  │  │  │  (500A)  │    │  (100Ω)  │    │  (50A)   │              │   │  │
│  │  │  └──────────┘    └──────────┘    └──────────┘              │   │  │
│  │  │                                                            │   │  │
│  │  │  ┌──────────┐                                              │   │  │
│  │  │  │ Contactor│    ┌──────────────┐  ┌──────────────┐        │   │  │
│  │  │  │    K3    │    │ Hall Effect  │  │   Voltage    │        │   │  │
│  │  │  │  DC- Main│    │   Current    │  │   Sensor     │        │   │  │
│  │  │  │  (500A)  │    │   Sensor     │  │   Module     │        │   │  │
│  │  │  └──────────┘    └──────────────┘  └──────────────┘        │   │  │
│  │  │                                                            │   │  │
│  │  │  ┌───────┐   ┌───────┐   ┌──────────────────────────┐     │   │  │
│  │  │  │ Fuse  │   │ Fuse  │   │   Discharge Resistor R2  │     │   │  │
│  │  │  │  F1   │   │  F2   │   │       (10kΩ, 100W)       │     │   │  │
│  │  │  │ DC+   │   │ DC-   │   └──────────────────────────┘     │   │  │
│  │  │  └───────┘   └───────┘                                    │   │  │
│  │  │                                                            │   │  │
│  │  └────────────────────────────────────────────────────────────┘   │  │
│  │                                                                  │  │
│  │  ══════════════════════════════════════════════════════════════  │  │
│  │  DC Output Terminal Block TB2                                    │  │
│  │  ┌──────┬──────┬──────┐                                          │  │
│  │  │ DC+  │ DC-  │  PE  │ ──────► To Charging Cable / Connector   │  │
│  │  └──────┴──────┴──────┘                                          │  │
│  │                                                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Mounting and Clearance Requirements

| Component | Mounting | Hardware | Clearance |
|-----------|----------|----------|-----------|
| K2 (DC+ contactor) | Bolt-down to panel | M8 × 4, 12 Nm | 30mm to adjacent components |
| K3 (DC- contactor) | Bolt-down to panel | M8 × 4, 12 Nm | 30mm to adjacent components |
| K4 (pre-charge) | DIN rail or bolt-down | M6 × 4, 5 Nm | 20mm clearance |
| R1 (pre-charge resistor) | Chassis mount (thermal pad) | M5 × 2, 3 Nm | 50mm from other components (heat) |
| R2 (discharge resistor) | Chassis mount | M5 × 2, 3 Nm | 30mm clearance |
| CB1 (DC breaker) | Bolt-down to rear panel | M8 × 4, 15 Nm | Per manufacturer spec |
| F1, F2 (fuses) | Bolt-in fuse holder | M10, 25 Nm | Per manufacturer spec |
| Current sensor | Clamp-on or bolt-through | — | Centered on conductor |
| Voltage sensor | DIN rail mount | — | Short leads to measurement point |

### 8.3 Wiring Within the Assembly

| Connection | Wire | Size | Color | Torque |
|------------|------|------|-------|--------|
| CB1 → K2 input | XLPE | 95 mm² | Red | 25 Nm |
| CB1 → R1 input | XLPE | 4 mm² | Red | 3 Nm |
| R1 → K4 input | XLPE | 4 mm² | Red | 3 Nm |
| K4 output → K2 output bus | XLPE | 4 mm² | Red | 3 Nm |
| K2 output → F1 | XLPE | 95 mm² | Red | 25 Nm |
| DC- in → K3 input | XLPE | 95 mm² | Black | 25 Nm |
| K3 output → F2 | XLPE | 95 mm² | Black | 25 Nm |
| F1 → TB2 DC+ | XLPE | 95 mm² | Red | 25 Nm |
| F2 → TB2 DC- | XLPE | 95 mm² | Black | 25 Nm |
| R2 across TB2 DC+/DC- | PVC | 1.5 mm² | Brown | 2 Nm |
| Coil wires (K2, K3, K4) | PVC screened | 1.0 mm² | Yellow | 1.5 Nm |
| Aux feedback wires | PVC screened | 0.75 mm² | White | 1.0 Nm |
| Current sensor output | Screened pair | 0.5 mm² | — | — |
| Voltage sensor leads | Screened pair | 0.5 mm² | — | — |

## 9. Thermal Considerations

### 9.1 Heat Generation

| Component | Heat Source | Typical Dissipation |
|-----------|-----------|---------------------|
| K2 contactor | Contact resistance (0.5 mΩ × 500A²) | 125W at full current |
| K3 contactor | Contact resistance | 125W at full current |
| K4 contactor | Only during pre-charge (<1 s) | Negligible steady-state |
| R1 pre-charge | Only during pre-charge (<1 s) | 0W steady-state, ~1 kJ per event |
| R2 discharge | Continuous when voltage present | 100W max (1000V), 16W typical (400V) |
| CB1 | Internal resistance | ~30W at full current |
| F1, F2 | Internal resistance | ~20W each at full current |
| Current sensor | Internal dissipation | ~5W |
| **Total** | | **~460W at 500A** |

### 9.2 Cooling Strategy

- Contactors K2 and K3 are positioned in the **liquid-cooled zone** (Zone 2), with airflow from the HVAC clip-on passing over their heat sinks
- R1 is chassis-mounted to the cabinet metalwork, using the cabinet wall as a heat spreader
- R2 is positioned with adequate clearance; its 100W continuous dissipation is manageable with natural convection
- Temperature monitoring via NTC thermistor near K2/K3 (connected to safety supervisor AI2)

## 10. Bill of Materials

| Item | Qty | Specification | Reference |
|------|-----|---------------|-----------|
| DC contactor 500A/1000V DC | 2 | K2, K3 — with 2NO+2NC aux | e.g., Gigavac GX26 or TE EV200 |
| DC contactor 50A/1000V DC | 1 | K4 — pre-charge, with 1NO aux | e.g., Gigavac GX14 or TE EV100 |
| Pre-charge resistor 100Ω/100W | 1 | R1 — wirewound, 1500J pulse | e.g., Vishay RPS or Ohmite |
| Discharge resistor 10kΩ/100W | 1 | R2 — wirewound, 1200V rated | e.g., Vishay RH series |
| DC circuit breaker 500A/1000V | 1 | CB1 — thermal-magnetic, with aux | e.g., ABB Tmax or Schneider NSX |
| Semiconductor fuse 500A/1200V gR | 2 | F1, F2 — ultra-fast | e.g., Bussmann FWH or Mersen |
| Fuse holder (bolt-in) | 2 | For F1, F2 | Matched to fuse type |
| Hall effect current sensor ±600A | 1 | Closed-loop, 4-20mA output | e.g., LEM DHAB or Honeywell |
| Voltage sensor module 0-1200V | 1 | Isolated, 0-10V output | e.g., LEM LV25-P or custom |
| Flyback TVS diodes | 3 | For K2, K3, K4 coils | 48V clamp, bidirectional |
| MOSFET coil driver boards | 3 | Isolated gate drive, PWM capable | Custom or off-shelf |
| Terminal block TB2 (DC output) | 1 | 3-position, 95mm² rated, 1000V | e.g., Phoenix Contact PTPOWER |
| Mounting hardware | 1 set | M8, M6, M5 bolts, Belleville washers | Grade 8.8 |
| Busbar links (internal) | 4 | 95mm² rated copper, tin-plated | Custom cut |

## 11. Testing and Commissioning

### 11.1 Factory Acceptance Tests (FAT)

| Test | Method | Pass Criteria |
|------|--------|---------------|
| Insulation resistance | 1000V DC megger, all terminals to PE | >10 MΩ |
| Contact resistance (K2, K3) | Micro-ohmmeter, closed contacts | <0.5 mΩ each |
| Pre-charge timing | Oscilloscope on V_output, close K4 into 1000µF load | 95% in <500 ms |
| Pre-charge resistor value | LCR meter, cold | 100Ω ±5% |
| Discharge time | Close contactors, charge to 1000V, open, monitor decay | <60V in 30 s |
| Contactor response time | Current probe on coil, voltage probe on output | Close <50 ms, Open <30 ms |
| Auxiliary feedback | Cycle each contactor, verify DI state change | Correct for all states |
| Weld detection | Simulate stuck feedback, verify fault code | F05 latched within 100 ms |
| Coil PWM hold | Thermal camera on coil after 30 min at hold current | Coil temp <80°C |
| Voltage sensor calibration | Apply known voltages (0, 200, 500, 800, 1000V) | ±0.5% across range |
| Current sensor calibration | Pass known currents (0, 100, 300, 500A) | ±0.5% across range |

### 11.2 Commissioning Checklist

- [ ] All bolted connections torqued to specification with torque markers applied
- [ ] Contactor coil wiring polarity verified
- [ ] Auxiliary feedback wiring verified (NC = closed when contactor open)
- [ ] Pre-charge resistor firmly mounted with thermal compound
- [ ] Discharge resistor clearance verified (no nearby combustibles)
- [ ] Fuse ratings verified against design (F1: 500A gR, F2: 500A gR)
- [ ] CB1 trip settings verified (magnetic and thermal)
- [ ] Current sensor zero offset calibrated
- [ ] Voltage sensor scaling verified against DMM reference
- [ ] Safety supervisor firmware version confirmed
- [ ] Full pre-charge cycle test with load bank completed
- [ ] Weld detection test completed (simulated fault)
- [ ] All auxiliary feedback signals logged and verified in ECU diagnostics

## 12. References

- IEC 61851-23: DC-specific requirements for EV charging
- IEC 60947-4-1: Contactors and motor-starters
- IEC 60269-4: Semiconductor fuses (gR class)
- IEC 62477-1: Safety requirements for power electronics
- [[01 - Hardware Components]] — Full component specifications
- [[02 - Electric Wiring Diagram]] — System-level wiring schematic
- [[04 - Backplane Power Management]] — Power distribution to this assembly
- [[docs/Software/03 - Safety Supervisor Controller|03 - Safety Supervisor Controller]] — Firmware controlling this assembly
- [[research/01 - Safety Philosophy|01 - Safety Philosophy]] — Interlock chain design

---

**Document Version**: 1.0
**Last Updated**: 2026-02-26
**Prepared by**: Power Electronics Engineering
