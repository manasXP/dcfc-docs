# CCS Connector and Liquid-Cooled Cable Assembly

Tags: #dcfc #hardware #connector #ccs #cable #liquid-cooling #charging

Related: [[01 - Hardware Components]] | [[02 - Electric Wiring Diagram]] | [[03 - Cabinet Layout]] | [[05 - DC Output Contactor and Pre-Charge Circuit]]

## 1. Overview

The CCS connector and liquid-cooled cable assembly is the physical interface between the DCFC and the electric vehicle. It carries high-voltage DC power (200–1000V, up to 500A), communication signals (Control Pilot, Proximity Pilot), protective earth, and a closed-loop coolant circuit — all within a single cable bundle that must be lightweight, flexible, and safe for daily public handling.

At power levels above 150 kW, the DC conductors would need to be impractically thick and heavy if air-cooled (>120 mm² per conductor). Liquid cooling allows the use of smaller conductors (35–50 mm²) by actively removing I²R heat, resulting in a cable that is roughly **half the weight and diameter** of an equivalent air-cooled assembly while supporting 2–3× the current.

## 2. System Context

```
    INSIDE CABINET                        CABLE ASSEMBLY                      EV INLET

    DC Output TB2 ─────────┐
    ┌──────────────────┐    │    ┌──────────────────────────────────────┐
    │  DC+ (from F1)   │────┼───►│  DC+ Conductor (35-50mm²)           │
    │  DC- (from F2)   │────┼───►│  DC- Conductor (35-50mm²)           │
    │  PE              │────┼───►│  PE Conductor (16mm²)               │
    └──────────────────┘    │    │                                      │
                            │    │  Coolant Supply (silicone tube)  ───►│──┐
    Coolant Manifold ───────┼───►│  Coolant Return (silicone tube) ◄───│──┤
                            │    │                                      │  │
    CP Circuit ─────────────┼───►│  CP Signal Wire (2.5mm²)            │  │
    PP Circuit ─────────────┼───►│  PP Signal Wire (1.5mm²)            │  │
                            │    │                                      │  │
    PLC Modem ──────────────┼───►│  PLC coupled on CP line             │  │
                            │    │                                      │  │
    Cable Temp Sensor ◄─────┼────│  NTC Thermistor (in cable)          │  │
                            │    │                                      │  │
                            │    │  Outer Jacket (TPE/PUR, 40-50mm OD) │  │
                            │    └──────────────────────────────────────┘  │
                            │                                              │
                            │    ┌──────────────────────────────────────┐  │
                            │    │         CCS CONNECTOR HEAD          │  │
                            │    │                                      │  │
                            └───►│  DC+ Pin ──────────────────► To EV  │  │
                                 │  DC- Pin ──────────────────► To EV  │  │
                                 │  PE  Pin ──────────────────► To EV  │  │
                                 │  CP  Pin ──────────────────► To EV  │  │
                                 │  PP  Pin ──────────────────► To EV  │  │
                                 │                                      │  │
                                 │  Coolant turnaround loop ◄──────────┼──┘
                                 │  (inside connector housing)          │
                                 │                                      │
                                 │  Temperature sensor (at pins)       │
                                 │  Locking actuator (solenoid/motor)  │
                                 │                                      │
                                 └──────────────────────────────────────┘
```

## 3. CCS Connector Standards

### 3.1 CCS Combo Variants

The Combined Charging System combines AC and DC pins in a single connector. Two regional variants exist:

| Parameter | CCS Combo 1 (CCS1) | CCS Combo 2 (CCS2) |
|-----------|---------------------|---------------------|
| Standard | SAE J1772 Combo / IEC 62196-3 Type 1 | IEC 62196-3 Type 2 |
| Region | North America, Japan, Korea | Europe, rest of world |
| AC pins | 5 pins (L1, L2/N, PE, CP, PP) | 7 pins (L1, L2, L3, N, PE, CP, PP) |
| DC pins | 2 pins (DC+, DC-) | 2 pins (DC+, DC-) |
| Total pins | 7 | 9 |
| AC charging | Up to 19.2 kW (Level 2) | Up to 43 kW (3-phase) |
| DC charging | Up to 500A, 1000V DC | Up to 500A, 1000V DC |
| Mechanical coding | Asymmetric keying | Asymmetric keying |

### 3.2 Pin Assignment (CCS2)

```
              CCS COMBO 2 CONNECTOR FACE
              (Vehicle inlet perspective)

         ┌─────────────────────────────────┐
         │                                 │
         │      AC SECTION (Type 2)        │
         │                                 │
         │    ○ L1        ○ L2             │
         │                                 │
         │         ○ PE                    │
         │                                 │
         │    ○ N         ○ L3             │
         │                                 │
         │    ○ CP        ○ PP             │
         │                                 │
         ├─────────────────────────────────┤
         │                                 │
         │      DC SECTION                 │
         │                                 │
         │       ◉ DC+       ◉ DC-        │
         │    (large pins, 500A rated)     │
         │                                 │
         └─────────────────────────────────┘
```

### 3.3 Pin Specifications

| Pin | Function | Diameter | Current Rating | Voltage Rating |
|-----|----------|----------|----------------|----------------|
| DC+ | Positive power delivery | 6.5 mm | 500A (liquid-cooled) / 200A (air-cooled) | 1000V DC |
| DC- | Negative power return | 6.5 mm | 500A / 200A | 1000V DC |
| PE | Protective earth | 4.0 mm | Per ground fault current | — |
| CP | Control Pilot | 1.0 mm | Signal only (±12V, 1 kHz PWM) | ±12V |
| PP | Proximity Pilot | 1.0 mm | Signal only | 0–5V |
| L1–L3, N | AC power (unused during DC charge) | 4.0 mm | 63A (AC mode) | 480V AC |

### 3.4 Connector Mechanical Requirements

| Parameter | Specification | Standard |
|-----------|---------------|----------|
| Mating cycles | ≥10,000 | IEC 62196-3 |
| Insertion force | <80 N | IEC 62196-3 |
| Extraction force (unlocked) | <80 N | IEC 62196-3 |
| Extraction force (locked) | >500 N (must not release) | IEC 62196-3 |
| Drop test | 1 m onto concrete, 3 orientations | IEC 62196-3 |
| UV resistance | 1000 hours UV exposure | IEC 62196-3 |
| Operating temperature | -40°C to +50°C ambient | IEC 62196-3 |
| IP rating (mated) | IP54 | IEC 62196-3 |
| IP rating (unmated) | IP44 (with cap) | IEC 62196-3 |
| Weight (connector head) | ≤1.5 kg (liquid-cooled) | Ergonomic target |

## 4. Liquid-Cooled Cable Design

### 4.1 Why Liquid Cooling Is Necessary

The DC conductors generate heat from I²R losses. At high currents, the heat generation in an air-cooled cable exceeds the cable's ability to dissipate it to ambient, causing the conductor temperature to exceed insulation limits.

**Comparison at 500A continuous:**

| Parameter | Air-Cooled Cable | Liquid-Cooled Cable |
|-----------|------------------|---------------------|
| Conductor size needed | 120 mm² (each DC+/DC-) | 35–50 mm² (each) |
| Cable outer diameter | 55–65 mm | 35–45 mm |
| Cable weight (per meter) | ~8 kg/m | ~3 kg/m |
| Cable weight (5 m assembly) | ~40 kg | ~15 kg |
| Max continuous current | 200A (at 90°C conductor) | 500A (at 90°C conductor) |
| Conductor temperature rise | Self-limiting at ~200A | Actively managed by coolant |
| Flexibility | Poor (very stiff) | Good (thinner conductors) |
| User handling | Difficult (heavy, rigid) | Manageable (lighter, flexible) |

At 500A through a 50 mm² copper conductor, the heat generated per meter is:

```
P = I² × R

Copper resistivity at 90°C: ρ ≈ 2.2 × 10⁻⁸ Ω·m
R per meter (50 mm²): R = ρ × L / A = 2.2e-8 × 1 / 50e-6 = 0.44 mΩ/m

Per conductor: P = 500² × 0.00044 = 110 W/m
Both conductors (DC+ and DC-): 220 W/m
For 5 m cable: 1100 W total heat generation
```

This 1.1 kW of heat must be continuously removed by the coolant flow.

### 4.2 Cable Cross-Section

```
LIQUID-COOLED CABLE CROSS-SECTION (Not to Scale)

                    ┌─── Outer Jacket (TPE/PUR, 3mm wall)
                    │
                    │   ┌─── Coolant Return Tube (silicone, 8mm ID)
                    │   │
                    │   │   ┌─── DC+ Conductor (50mm², stranded Cu)
                    │   │   │       with XLPE insulation (1.5kV rated)
                    │   │   │
                    ▼   ▼   ▼
              ┌─────────────────────────┐
              │   ╭─────╮               │
              │   │Cool │   ╭─────╮     │
              │   │Ret. │   │ DC+ │     │
              │   ╰─────╯   ╰─────╯     │
              │                         │
              │   ╭─────╮   ╭─────╮     │
              │   │ DC- │   │Cool │     │
              │   ╰─────╯   │Sup. │     │
              │             ╰─────╯     │
              │                         │
              │   ╭───╮ ╭───╮ ╭───╮     │
              │   │PE │ │CP │ │PP │     │
              │   ╰───╯ ╰───╯ ╰───╯     │
              │                         │
              │     ╭───────────╮       │
              │     │Temp Sensor│       │
              │     │  (NTC)   │       │
              │     ╰───────────╯       │
              │                         │
              └─────────────────────────┘

              ◄────── 40-45 mm OD ──────►
```

### 4.3 Cable Component Specifications

#### DC Power Conductors

| Parameter | Value | Notes |
|-----------|-------|-------|
| Material | Electrolytic copper, Class 6 (finest stranding) | Maximum flexibility |
| Cross-section | 50 mm² per conductor | 2 conductors: DC+ and DC- |
| Stranding | >1000 strands of 0.2 mm wire | Achieves 15 mm minimum bend radius |
| Insulation | XLPE (cross-linked polyethylene) | 1.5 kV rated, 105°C continuous |
| Insulation thickness | 1.5 mm | Per IEC 60502 |
| DC resistance at 20°C | 0.387 mΩ/m | Per IEC 60228 Class 6 |
| DC resistance at 90°C | 0.50 mΩ/m | Temperature-corrected |
| Current rating (liquid-cooled) | 500A continuous | At 60°C coolant, 90°C conductor |

#### Coolant Tubes

| Parameter | Value | Notes |
|-----------|-------|-------|
| Material | Reinforced silicone or EPDM | Chemical resistant, flexible |
| Inner diameter | 8 mm | Supply and return |
| Wall thickness | 2 mm | Rated for 5 bar burst pressure |
| Working pressure | 1–3 bar | Per cooling system spec |
| Temperature rating | -40°C to +150°C | Silicone grade |
| Flow rate (per cable) | 2–5 L/min | Sufficient for 1.1 kW heat removal |
| Coolant type | 50/50 ethylene glycol-water | Same as cabinet cooling loop |
| Fittings | Quick-disconnect at cabinet penetration | Push-to-connect or JIC fittings |

#### Signal Wires

| Wire | Size | Insulation | Purpose |
|------|------|------------|---------|
| CP (Control Pilot) | 2.5 mm² | XLPE, shielded | IEC 61851 PWM signal + ISO 15118 PLC |
| PP (Proximity Pilot) | 1.5 mm² | XLPE | Cable detection + current rating coding |
| PE (Protective Earth) | 16 mm² | PVC, green/yellow | Safety ground per IEC 62196 |
| Temperature sensor | 0.5 mm² × 2 (pair) | PFA, shielded | NTC thermistor in cable and connector |

#### Outer Jacket

| Parameter | Value | Notes |
|-----------|-------|-------|
| Material | TPE (Thermoplastic Elastomer) or PUR (Polyurethane) | UV resistant, flame retardant |
| Outer diameter | 40–45 mm | Comfortable hand grip |
| Wall thickness | 3 mm | Abrasion and crush resistant |
| Flame retardant | IEC 60332-1-2 | Self-extinguishing |
| Oil resistance | EN 60811-404 | Fuel and oil splash |
| Color | Black or dark gray | UV stable |
| Temperature rating | -40°C to +90°C | Outdoor year-round operation |
| Bend radius (minimum) | 150 mm (6× OD) | For repeated flexing |
| Crush resistance | 2000 N/100 mm | Per EN 50306 |

### 4.4 Cable Assembly Specifications

| Parameter | Value |
|-----------|-------|
| Total cable length | 4–5 m (standard) |
| Cable weight (complete assembly) | ~3 kg/m (~15 kg total for 5 m) |
| Connector head weight | ~1.5 kg |
| Total assembly weight | ~16.5 kg (5 m cable + connector) |
| Minimum bend radius (installed) | 200 mm |
| Minimum bend radius (flexing) | 150 mm |
| Tensile strength (cable) | ≥2000 N |
| Pull force at connector strain relief | ≥1000 N |
| Design life | ≥10 years or 100,000 charging sessions |

## 5. Coolant Circuit Integration

### 5.1 Cable Coolant Loop

The liquid-cooled cable is part of the charger's overall thermal management coolant circuit. The cable coolant loop is a dedicated branch with its own flow control, sharing the same coolant pump and reservoir as the power module cooling.

```
COOLANT CIRCUIT (Cable Branch)

    ┌─── COOLANT PUMP (24V DC, from PDU 3)
    │
    ├─── RESERVOIR (2L expansion tank, with level sensor)
    │
    ├─── POWER MODULE BRANCH
    │    Module 1 → Module 2 → Return
    │
    └─── CABLE COOLING BRANCH
         │
         ├─── Flow Control Valve (optional, solenoid)
         │    Opens when charging session active
         │
         ├─── Flow Sensor (turbine or paddle wheel)
         │    Output: Pulse to Main ECU
         │    Purpose: Verify flow before enabling high current
         │
         ├─── CABLE SUPPLY (coolant in)
         │    │
         │    ├─── Runs along DC+ conductor (absorbs heat)
         │    │
         │    ├─── Through connector head (cools pins)
         │    │
         │    └─── TURNAROUND in connector housing
         │
         ├─── CABLE RETURN (coolant out)
         │    │
         │    └─── Runs along DC- conductor (absorbs heat)
         │
         ├─── Temperature Sensor T_cable_out
         │    NTC 10kΩ at return line
         │    Connected to: Main ECU AI4
         │
         └─── Returns to reservoir → pump → radiator
```

### 5.2 Coolant Flow Requirements

| Parameter | Value | Notes |
|-----------|-------|-------|
| Minimum flow rate (per cable) | 2 L/min | At currents ≤250A |
| Nominal flow rate | 3.5 L/min | At 500A continuous |
| Maximum flow rate | 5 L/min | Safety margin / high ambient |
| Coolant inlet temperature | 30–45°C | Depends on ambient and radiator |
| Coolant outlet temperature (cable) | 45–60°C | ΔT ≈ 15°C at 500A |
| Maximum coolant temperature | 65°C | Above this → derate current |
| Pressure drop through cable | 0.3–0.8 bar | At 3.5 L/min, 5 m length |

### 5.3 Heat Removal Calculation

```
Heat generated in cable at 500A (both conductors, 5 m):
  Q_gen = 1100 W (from Section 4.1)

Heat removed by coolant:
  Q = ṁ × Cp × ΔT

  Where:
    ṁ  = mass flow rate = 3.5 L/min × 1.06 kg/L = 3.71 kg/min = 0.062 kg/s
    Cp = specific heat of 50/50 glycol-water = 3.4 kJ/(kg·°C)
    ΔT = temperature rise through cable

  Solving for ΔT:
    ΔT = Q / (ṁ × Cp) = 1100 / (0.062 × 3400) = 5.2°C

  At 3.5 L/min, the coolant rises only 5.2°C through the cable.
  This provides substantial margin — even at 2 L/min minimum:
    ΔT = 1100 / (0.035 × 3400) = 9.2°C (still well within limits)
```

### 5.4 Coolant Quick-Disconnect at Cabinet

The coolant supply and return lines connect to the cabinet's coolant manifold via quick-disconnect fittings at the cable entry point. This allows cable replacement without draining the entire coolant system.

| Parameter | Value |
|-----------|-------|
| Fitting type | Non-spill quick-disconnect (e.g., CPC or Staubli) |
| Size | 3/8" (10 mm) |
| Material | Brass with EPDM seals |
| Spill volume on disconnect | <1 mL |
| Pressure rating | 5 bar |
| Automatic shut-off | Both halves close on disconnect |

## 6. Connector Head Design

### 6.1 Internal Layout

```
CCS CONNECTOR HEAD (Cut-Away, Side View)

    ┌──────────────────────────────────────────────────────────┐
    │                                                          │
    │   CABLE ENTRY (rear)                                     │
    │   ┌──────────────┐                                       │
    │   │ Strain Relief│  Compression gland + flex boot        │
    │   │ + Cable Gland│  (IP67 when properly assembled)       │
    │   └──────┬───────┘                                       │
    │          │                                               │
    │   ┌──────┴───────────────────────────────────────────┐   │
    │   │                                                  │   │
    │   │   Coolant Supply ──────────────────┐             │   │
    │   │                                    │             │   │
    │   │   DC+ Conductor ──► DC+ Pin        │             │   │
    │   │                     (crimped +     │             │   │
    │   │                      bolted)       │             │   │
    │   │                                    │             │   │
    │   │   DC- Conductor ──► DC- Pin        │             │   │
    │   │                                    │             │   │
    │   │   Coolant Supply ──► Cooling       │             │   │
    │   │                      jacket ───────┤             │   │
    │   │                      around        │             │   │
    │   │                      DC pins ◄─────┘             │   │
    │   │                      │                           │   │
    │   │   Coolant Return ◄───┘                           │   │
    │   │                                                  │   │
    │   │   ┌────────────────────────────────────────┐     │   │
    │   │   │  TEMPERATURE SENSOR (NTC)              │     │   │
    │   │   │  Bonded to DC+ pin base                │     │   │
    │   │   │  Measures pin/contact temperature      │     │   │
    │   │   └────────────────────────────────────────┘     │   │
    │   │                                                  │   │
    │   │   ┌────────────────────────────────────────┐     │   │
    │   │   │  LOCKING MECHANISM                     │     │   │
    │   │   │  Motorized actuator (12V DC)           │     │   │
    │   │   │  or solenoid latch                     │     │   │
    │   │   │  Feedback: Microswitch (lock status)   │     │   │
    │   │   └────────────────────────────────────────┘     │   │
    │   │                                                  │   │
    │   └──────────────────────────────────────────────────┘   │
    │                                                          │
    │   MATING FACE (front) ──► To EV inlet                    │
    │   ┌──────────────────────────────────────────────────┐   │
    │   │  AC Pins (L1, L2, L3, N, PE, CP, PP)            │   │
    │   │  DC Pins (DC+, DC-)                              │   │
    │   │  IP54 gasket seal                                │   │
    │   └──────────────────────────────────────────────────┘   │
    │                                                          │
    │   ERGONOMIC GRIP HOUSING (overmolded TPE)                │
    │   - Trigger button (unlock request)                      │
    │   - LED ring (status indication)                         │
    │   - Weight: ≤1.5 kg                                      │
    │                                                          │
    └──────────────────────────────────────────────────────────┘
```

### 6.2 Connector Pin Cooling

Inside the connector head, the coolant flows through a machined aluminum or copper cooling jacket that wraps around the DC+ and DC- pin barrels. This is critical because the pin-to-socket contact interface generates additional heat beyond the cable I²R losses:

```
Pin contact heat generation:
  Contact resistance (new, well-mated): ~50 µΩ per pin
  Contact resistance (worn, 10k cycles): ~200 µΩ per pin

  At 500A:
    P_contact = I² × R = 500² × 200e-6 = 50W per pin
    Both pins: 100W

  Combined with cable entry heat: ~100W + cable thermal load
  → Cooling jacket must absorb ~200W within the connector head
```

The coolant enters the connector, wraps around both DC pin barrels in a serpentine channel, and exits through the return tube. This keeps the pin surface temperature below 90°C even at 500A.

### 6.3 Temperature Monitoring

Two temperature sensors are integrated into the connector assembly:

| Sensor | Location | Type | Range | Connected To | Purpose |
|--------|----------|------|-------|--------------|---------|
| T_pin | Bonded to DC+ pin barrel inside connector | NTC 10kΩ | -40 to +200°C | Main ECU AI (via cable) | Over-temperature protection at contact interface |
| T_cable | Embedded in cable mid-length | NTC 10kΩ | -40 to +150°C | Main ECU AI4 | Cable conductor temperature monitoring |

**Over-temperature response:**

| Temperature | Action |
|-------------|--------|
| T_pin < 70°C | Normal operation, full current allowed |
| T_pin 70–80°C | Warning logged, increase coolant flow rate |
| T_pin 80–90°C | Derate charging current by 25% |
| T_pin 90–100°C | Derate charging current by 50% |
| T_pin > 100°C | Terminate charging session, open contactors |
| T_cable > 90°C | Derate by 25%, check coolant flow |
| T_cable > 105°C | Terminate session |

### 6.4 Connector Locking Mechanism

IEC 62196-3 requires that the connector cannot be withdrawn while DC voltage is present at the pins. The charger must verify that the connector is locked before closing the DC output contactors.

| Parameter | Value |
|-----------|-------|
| Lock type | Motorized pin lock or solenoid latch |
| Actuator voltage | 12V DC |
| Actuator current | 0.5–2A (momentary) |
| Lock engagement time | <500 ms |
| Unlock engagement time | <500 ms |
| Holding force (locked) | ≥500 N axial pull |
| Manual emergency release | Mechanical override accessible to service personnel |
| Feedback | Microswitch — closed = locked, open = unlocked |
| Feedback connected to | Safety Supervisor DI7 (connector latch status) |

**Lock sequence:**
```
1. User inserts connector → PP resistance detected (cable present)
2. CP signal establishes → IEC 61851 state machine begins
3. CM5 commands lock actuator → lock engages
4. Lock microswitch confirms LOCKED → reported to safety supervisor
5. Safety supervisor allows DC contactor closure only if DI7 = LOCKED
6. ...charging session...
7. Session ends → current ramped to 0 → contactors open
8. Voltage at pins < 60V confirmed (discharge resistor + power module off)
9. CM5 commands unlock → lock disengages
10. User can withdraw connector
```

### 6.5 LED Status Ring

An LED ring around the connector handle provides user-visible charging status:

| Color | Pattern | Meaning |
|-------|---------|---------|
| White | Steady | Charger available, ready to connect |
| Blue | Pulsing | Communication established, authenticating |
| Green | Pulsing | Charging in progress |
| Green | Steady | Charging complete |
| Red | Steady | Fault — do not remove connector |
| Red | Flashing | Emergency — connector locked, contact operator |
| Off | — | Charger powered off |

| Parameter | Value |
|-----------|-------|
| LED type | RGB LED strip (addressable, WS2812 or similar) |
| Voltage | 5V DC (via signal cable from cabinet) |
| Current | <500 mA |
| Brightness | 200+ lumen (visible in daylight) |
| IP rating | IP67 (potted/sealed in connector housing) |

## 7. Control Pilot and Proximity Pilot Circuits

### 7.1 Control Pilot (CP)

The CP pin carries a ±12V, 1 kHz PWM signal from the EVSE to the vehicle. The duty cycle communicates the maximum available current. During DC charging, the CP line also carries ISO 15118 PLC (Power Line Communication) for high-level communication.

```
                CHARGER SIDE                    EV SIDE

    ┌────────────────────────┐            ┌──────────────────┐
    │                        │            │                  │
    │  +12V ──┬── R1 (1kΩ) ──┼──── CP ────┼──── R_vehicle ───┤
    │         │              │    wire    │      (to GND)    │
    │  PWM ───┘              │            │                  │
    │  Generator             │            │  R_vehicle value │
    │  (1 kHz)               │            │  determines      │
    │                        │            │  CP voltage:     │
    │  ADC ◄── Voltage ──────┤            │   2.74kΩ = 9V   │
    │          Monitor       │            │   882Ω = 6V     │
    │                        │            │   246Ω = 3V     │
    │  -12V ─────────────────┤            │                  │
    │                        │            │                  │
    │  PLC Modem ◄──►        │            │                  │
    │  (HomePlug GreenPHY)   │            │                  │
    │  coupled via           │            │                  │
    │  transformer on CP     │            │                  │
    │                        │            │                  │
    └────────────────────────┘            └──────────────────┘
```

**CP States (IEC 61851-1):**

| State | CP Voltage (positive half) | Meaning |
|-------|---------------------------|---------|
| A | +12V (no load) | No vehicle connected |
| B | +9V (2.74 kΩ) | Vehicle connected, not ready |
| C | +6V (882 Ω) | Vehicle ready, charging requested |
| D | +3V (246 Ω) | Vehicle ready, ventilation required |
| E | 0V (short to PE) | Fault — CP shorted |
| F | -12V | EVSE not available |

During DC charging, the CP PWM duty cycle is set to 5% (per IEC 61851-23), indicating "digital communication required." The actual current negotiation happens via ISO 15118 PLC on the same CP wire.

### 7.2 Proximity Pilot (PP)

The PP pin detects cable connection and communicates the cable's maximum current rating through a resistor coded into the cable plug.

| PP Resistor Value | Cable Current Rating | Voltage at EVSE |
|-------------------|---------------------|-----------------|
| 1.5 kΩ | 13A | ~4.1V |
| 680 Ω | 20A | ~3.1V |
| 220 Ω | 32A | ~1.8V |
| 100 Ω | 63A (or DC mode indicator) | ~0.9V |

For DC charging with CCS, the PP resistor in the DC section of the connector typically signals "DC mode" rather than a specific current limit — the actual current limit is negotiated digitally via ISO 15118.

**PP detection circuit in charger:**
```
    +5V
     │
     R_pull-up (1 kΩ)
     │
     ├──── To ADC (Main ECU) ──── Voltage reading
     │
     PP pin ──── R_cable (coded) ──── GND (in connector plug)
```

## 8. Cable Management and Holster

### 8.1 Cable Retractor / Holster Design

The cable assembly rests in a holster on the front panel of the cabinet when not in use. The holster must support the weight of the cable, protect the connector from damage, and allow easy retrieval by the user.

```
CABLE HOLSTER (Front Panel View)

    ┌──────────────────────────────────────────────┐
    │                                              │
    │   ┌──────────────────────────────────────┐   │
    │   │                                      │   │
    │   │        CABLE HOOK / CRADLE           │   │
    │   │     (supports cable weight)          │   │
    │   │                                      │   │
    │   │   ╭─────────╮  Cable loops           │   │
    │   │   │ Cable   │  around cradle         │   │
    │   │   │ loop 1  │  arms                  │   │
    │   │   ╰─────────╯                        │   │
    │   │                                      │   │
    │   │   ╭─────────╮                        │   │
    │   │   │ Cable   │                        │   │
    │   │   │ loop 2  │                        │   │
    │   │   ╰─────────╯                        │   │
    │   │                                      │   │
    │   └──────────────────────────────────────┘   │
    │                                              │
    │   ┌──────────────────────────────────────┐   │
    │   │                                      │   │
    │   │      CONNECTOR CRADLE                │   │
    │   │   ╭───────────────────────╮          │   │
    │   │   │                       │          │   │
    │   │   │   CCS Connector       │          │   │
    │   │   │   (resting position)  │          │   │
    │   │   │                       │          │   │
    │   │   ╰───────────────────────╯          │   │
    │   │                                      │   │
    │   │   - Angled for easy grab             │   │
    │   │   - Drip guard above                 │   │
    │   │   - Rain cap (auto-closing)          │   │
    │   │                                      │   │
    │   └──────────────────────────────────────┘   │
    │                                              │
    └──────────────────────────────────────────────┘
```

### 8.2 Strain Relief

| Parameter | Value |
|-----------|-------|
| Type | Multi-stage: internal compression gland + external flex boot |
| Pull resistance | ≥1000 N axial at cable entry to connector |
| Flex boot material | Silicone or EPDM, 80 Shore A |
| Flex boot length | 150 mm |
| Bend restriction | Prevents bends tighter than 150 mm radius at entry point |
| Cable entry to cabinet | Sealed gland (M50) with strain relief bracket rated for 5× cable weight |

### 8.3 Cable Entry to Cabinet

```
CABINET CABLE PENETRATION (Cross-Section)

    Cabinet Wall
    │
    │   ┌───────────────────────────────────────┐
    │   │  CABLE GLAND (M50, IP68)              │
    │   │                                       │
    │   │  ┌─── Outer seal (compression ring)   │
    │   │  │                                    │
    │   │  │  ┌─ Cable outer jacket             │
    │   │  │  │                                 │
    │   │  │  │  DC+, DC-, PE, CP, PP wires     │
    │   │  │  │  split to terminal block TB2    │
    │   │  │  │                                 │
    │   │  │  │  Coolant supply tube ──► to manifold quick-disconnect
    │   │  │  │  Coolant return tube ──► to manifold quick-disconnect
    │   │  │  │                                 │
    │   │  │  │  Temp sensor wire ──► to ECU AI │
    │   │  │  │                                 │
    │   │  └──┘                                 │
    │   │                                       │
    │   │  Strain relief bracket                │
    │   │  (bolted to cabinet, takes cable load)│
    │   │                                       │
    │   └───────────────────────────────────────┘
    │
```

The cable transitions from a single bundle outside the cabinet to individual conductors inside. The breakout point is at the strain relief bracket, immediately after the cable gland. Each conductor routes to its respective terminal or connection point:

| Conductor | Routes To | Connection |
|-----------|-----------|------------|
| DC+ | Terminal block TB2, pin 1 | Bolted lug, 25 Nm |
| DC- | Terminal block TB2, pin 2 | Bolted lug, 25 Nm |
| PE | Terminal block TB2, pin 3 (bonded to ground bar) | Bolted lug, 15 Nm |
| CP | CP circuit on EVSE aux board | Push-in terminal |
| PP | PP circuit on EVSE aux board | Push-in terminal |
| Coolant supply | Coolant manifold | Quick-disconnect fitting |
| Coolant return | Coolant manifold | Quick-disconnect fitting |
| Temp sensor (T_pin) | Main ECU analog input | Push-in terminal |
| Temp sensor (T_cable) | Main ECU analog input AI4 | Push-in terminal |
| LED power (5V + data) | LED driver on EVSE aux board | Push-in terminal |
| Lock actuator (12V) | Lock driver on EVSE aux board | Push-in terminal |
| Lock microswitch | Safety Supervisor DI7 | Push-in terminal |

## 9. Safety and Compliance

### 9.1 Safety Requirements per IEC 61851-23

| Requirement | Implementation |
|-------------|----------------|
| Galvanic isolation when unplugged | DC contactors open, discharge resistor bleeds voltage <60V |
| Connector locked during DC charging | Motorized lock, verified by safety supervisor before contactor closure |
| PE continuity verified before energizing | PE continuity check via CP/PP circuit before DC contactors close |
| Voltage at pins <60V before unlock | Safety supervisor confirms V_output < 60V (via discharge resistor R2) |
| IP protection during charging | IP54 mated, gasket seal between connector and vehicle inlet |
| Over-temperature protection | T_pin and T_cable sensors with derating and shutdown thresholds |
| Cable strain relief | Multi-stage relief rated for 5× cable weight |

### 9.2 Insulation and Creepage

| Parameter | Value | Standard |
|-----------|-------|----------|
| Conductor insulation voltage rating | 1.5 kV DC | IEC 60502 |
| Creepage distance (DC pins to PE) | ≥12.5 mm | IEC 62196-3 |
| Clearance distance (DC pins to PE) | ≥10 mm | IEC 62196-3 |
| Insulation resistance (cable) | >100 MΩ at 1000V DC | IEC 62196-3 |
| Hi-pot test (production) | 3.5 kV AC for 60 s, no breakdown | IEC 62196-3 |

### 9.3 Coolant Leak Safety

A coolant leak inside the connector or cable near high-voltage conductors is a safety concern. Mitigations:

| Risk | Mitigation |
|------|------------|
| Coolant contacts DC pins | Coolant is non-conductive glycol-water mix; pins are insulated from coolant jacket by XLPE barriers |
| Coolant leak inside connector | Drain holes in connector housing route leaked coolant to exterior, away from pins |
| Coolant tube rupture in cable | Cable jacket contains leak; flow sensor detects loss of flow → derate / shutdown |
| Coolant pressure loss | Flow sensor + pressure sensor detect anomaly within 1 s |
| External coolant drip | Drip guard on cable holster; condensate management |

## 10. Inspection and Maintenance

### 10.1 Routine Inspection Schedule

| Interval | Task | Criteria |
|----------|------|----------|
| Daily (automated) | CP/PP signal integrity check | CP voltage states correct, PP resistance valid |
| Daily (automated) | Coolant flow rate check (at session start) | ≥2 L/min before enabling high current |
| Daily (automated) | Lock mechanism cycle test | Lock engages and disengages within 500 ms |
| Weekly | Visual inspection of cable jacket | No cuts, abrasion, kinks, or crush damage |
| Weekly | Connector face inspection | No bent pins, debris, or moisture |
| Monthly | Connector mating contact resistance | <200 µΩ per pin (milliohm meter) |
| Monthly | Coolant line inspection at quick-disconnects | No seepage, fittings tight |
| Quarterly | Strain relief torque check | Per specification |
| Quarterly | Lock mechanism function test (manual) | Smooth engagement, no binding |
| Annually | Full cable insulation resistance test | >100 MΩ at 1000V DC |
| Annually | Coolant tube pressure test | Hold 3 bar for 10 min, no leak |
| Annually | Cable flexibility check | Bend through full range, no stiffness or cracking |

### 10.2 Wear Indicators

| Indicator | Sign of Wear | Action |
|-----------|-------------|--------|
| T_pin rising over time at same current | Pin contact degradation | Inspect pin surface, clean or replace connector |
| Contact resistance increasing | Pin wear or contamination | Clean contacts; replace connector if >500 µΩ |
| Coolant flow rate decreasing | Tube kink, blockage, or fitting leak | Inspect tubes, flush coolant circuit |
| Jacket abrasion visible | UV degradation, mechanical wear | Plan cable replacement |
| Lock engagement time increasing | Mechanism wear | Lubricate or replace lock actuator |
| PP resistance drifting | Cable plug resistor degradation | Replace PP resistor (if serviceable) or connector |

### 10.3 Cable Replacement

The cable assembly is a field-replaceable unit. Replacement time target: **1 hour** including functional verification.

```
CABLE REPLACEMENT PROCEDURE

1. Ensure charger is de-energized (main disconnect OFF)
2. Drain cable coolant branch:
   a. Close flow control valve (if installed)
   b. Disconnect quick-disconnect fittings (non-spill type, <1 mL loss)
3. Disconnect electrical:
   a. Remove DC+ and DC- lugs from TB2 (25 Nm bolts)
   b. Remove PE lug from TB2
   c. Disconnect CP, PP, temp sensor, LED, and lock wires from EVSE aux board
4. Remove cable gland and strain relief bracket
5. Withdraw cable assembly from cabinet
6. Install new cable assembly (reverse order)
7. Torque all connections per specification
8. Reconnect coolant lines, open flow valve
9. Bleed air from coolant lines (run pump for 2 min with contactors open)
10. Functional verification:
    a. Insulation resistance test (1000V megger, >100 MΩ)
    b. PE continuity (<0.1 Ω from cabinet ground bar to connector PE pin)
    c. CP/PP signal test (state transitions A→B→C)
    d. Coolant flow test (≥2 L/min, no leaks)
    e. Lock cycle test
    f. Test charge with EV or load bank
```

## 11. Bill of Materials

| Item | Qty | Specification | Notes |
|------|-----|---------------|-------|
| CCS2 connector head (liquid-cooled) | 1 | IEC 62196-3, 500A/1000V DC, with cooling jacket | e.g., Phoenix Contact, REMA, or ITT Cannon |
| Liquid-cooled cable assembly, 5 m | 1 | 2×50mm² DC + coolant tubes + signal wires | Pre-assembled, tested |
| Coolant quick-disconnect fitting (supply) | 1 | Non-spill, 3/8", brass/EPDM | CPC or Staubli |
| Coolant quick-disconnect fitting (return) | 1 | Non-spill, 3/8", brass/EPDM | CPC or Staubli |
| Cable gland (M50, IP68) | 1 | Nickel-plated brass | For cabinet wall penetration |
| Strain relief bracket | 1 | Steel, zinc-plated | Rated for 5× cable weight |
| Flex boot (connector end) | 1 | Silicone, 80 Shore A, 150 mm | Integrated in connector assembly |
| NTC 10kΩ temp sensor (T_pin) | 1 | Bonded inside connector head | Pre-installed in connector |
| NTC 10kΩ temp sensor (T_cable) | 1 | Embedded at cable mid-length | Pre-installed in cable |
| Lock actuator (motorized) | 1 | 12V DC, with microswitch feedback | Pre-installed in connector |
| LED ring (RGB addressable) | 1 | WS2812, IP67 potted | Pre-installed in connector |
| PP coding resistor | 1 | 100Ω (DC mode) or per cable rating | In connector plug body |
| Cable holster / cradle (cabinet-mounted) | 1 | Powder-coated steel, with drip guard | Supports full cable + connector weight |
| Rain cap (connector cradle) | 1 | TPE, spring-loaded auto-close | Protects unmated connector from rain |
| DC terminal lugs (50mm², M10 hole) | 2 | Crimp + bolted, tin-plated copper | For TB2 connection |
| PE terminal lug (16mm², M8 hole) | 1 | Crimp + bolted, tin-plated copper | For TB2 PE connection |

## 12. Dual-Connector Configuration (Optional)

For chargers with two charging outlets (power-shared), two complete cable assemblies are required. Each has its own:
- Set of DC contactors (K2a/K3a and K2b/K3b) — see [[05 - DC Output Contactor and Pre-Charge Circuit]]
- Coolant branch with independent flow sensor
- CP/PP circuit
- Lock mechanism and temperature sensors

Power sharing is managed by the CM5 energy manager, which dynamically allocates available power module output between the two outlets based on demand and priority.

```
                            POWER MODULES
                                │
                    ┌───────────┴───────────┐
                    │                       │
             ┌──────┴──────┐         ┌──────┴──────┐
             │ Contactor   │         │ Contactor   │
             │ Assembly A  │         │ Assembly B  │
             │ (K2a, K3a)  │         │ (K2b, K3b)  │
             └──────┬──────┘         └──────┬──────┘
                    │                       │
             ┌──────┴──────┐         ┌──────┴──────┐
             │ Cable       │         │ Cable       │
             │ Assembly 1  │         │ Assembly 2  │
             │ (CCS2)      │         │ (CCS2/NACS) │
             └─────────────┘         └─────────────┘
```

## 13. References

- IEC 62196-1: Plugs, socket-outlets, vehicle connectors — general requirements
- IEC 62196-3: Vehicle connectors — DC pins and contact-tubes
- IEC 61851-1: EV conductive charging — general requirements
- IEC 61851-23: DC-specific requirements
- ISO 15118: Vehicle-to-grid communication interface
- SAE J1772: SAE Electric Vehicle Conductive Charge Coupler
- SAE J3400: North American Charging Standard (NACS)
- [[01 - Hardware Components]] — Connector and cable overview
- [[02 - Electric Wiring Diagram]] — CP/PP circuit and DC output wiring
- [[03 - Cabinet Layout]] — Cable holster and entry point
- [[05 - DC Output Contactor and Pre-Charge Circuit]] — Upstream switching and protection

---

**Document Version**: 1.0
**Last Updated**: 2026-02-26
**Prepared by**: Connector & Cable Engineering
