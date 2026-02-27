# HVAC Unit Specification

Tags: #dcfc #hvac #specification #thermal #cooling

Related: [[docs/Hardware/06 - HVAC Clip-On Unit Hardware Design|06 - HVAC Clip-On Unit Hardware Design]] | [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]] | [[docs/Hardware/04 - Backplane Power Management|04 - Backplane Power Management]] | [[docs/System/01 - System Architecture|01 - System Architecture]] | [[docs/Components/10 - HVAC Clip-On Unit|10 - HVAC Clip-On Unit]]

---

## 1. Overview

The HVAC clip-on unit is a self-contained, externally mounted refrigeration system that maintains the sealed DCFC cabinet at safe operating temperatures. It creates a **closed-loop internal air circuit** — hot air is drawn from the cabinet, passed over a refrigeration evaporator, and returned as cooled air. External ambient air never contacts the electronics, enabling an IP55-rated sealed enclosure.

The unit is designed as a **field-replaceable unit (FRU)** that can be swapped in under 30 minutes without opening the main cabinet or disturbing power electronics. It mounts to the cabinet side, rear, or top via a standardised clip-on interface with quick-disconnect electrical, CAN, and condensate connections.

Two system variants are defined:

| Parameter | 150 kW Charger | 350 kW Charger |
|-----------|----------------|----------------|
| Required cooling capacity | ~9 kW (with 20% margin) | ~20 kW (with 20% margin) |
| Internal heat load | ~6.75 kW | ~15.5 kW |
| Solar + ambient ingress | ~0.75 kW | ~1.0 kW |

See [[docs/Hardware/06 - HVAC Clip-On Unit Hardware Design|06 - HVAC Clip-On Unit Hardware Design]] §3 for the full thermal load analysis.

---

## 2. Electrical Specification

### 2.1 Power Supply

The HVAC unit receives power through two independent feeds at the clip-on interface connector. The applicable configuration depends on the cooling capacity of the installed unit.

#### Configuration A — 24V DC Only (units ≤3 kW)

For small units (e.g. partial-load or mild-climate deployments), the HVAC runs entirely from the 24V DC bus supplied by PDU 3.

| Parameter | Value |
|-----------|-------|
| Supply | 24V DC from PDU 3 |
| Fuse | 6A (in PDU 3) |
| Max draw | ~400W |
| Loads | Controller, fans, brushless DC compressor inverter |

#### Configuration B — 400V AC + 24V DC (units 5–20 kW)

For the full-capacity units (9 kW / 20 kW), the compressor inverter requires a single-phase 400V AC feed tapped from the backplane (L1–L2, line-to-line). The 24V DC feed from PDU 3 serves as backup power for the controller, CAN interface, and fans.

| Parameter | 150 kW (9 kW unit) | 350 kW (20 kW unit) |
|-----------|---------------------|---------------------|
| AC supply | 400V AC, single-phase (L1–L2) | 400V AC, single-phase (L1–L2) |
| AC breaker | MCB 10A | MCB 16A |
| AC max current | 7A | 15A |
| 24V DC supply | From PDU 3, 6A fused | From PDU 3, 6A fused |
| Total electrical input | ~3 kW | ~7 kW |

The HVAC unit includes an internal buck converter (400V AC → 24V DC, 2A) so the controller and fans can run from the AC supply. The PDU 3 24V DC feed provides redundant power for the controller and CAN interface if the AC supply is lost.

> [!note] Backplane AC Feed
> The 400V AC L-L feed is routed from a dedicated busbar tap (TAP 5, L1+L2) through CB-HVAC (2-pole MCB) directly to the clip-on interface connector, bypassing PDU 3. See [[docs/Hardware/04 - Backplane Power Management|04 - Backplane Power Management]] §4.3 for the full circuit diagram. The 24V DC feed remains on PDU 3.

### 2.2 Electrical Protection (Within HVAC Unit)

| Device | Rating | Purpose |
|--------|--------|---------|
| Input fuse (AC) | 16A / 32A gG | Compressor overcurrent |
| Input fuse (24V DC) | 10A | Controller and fan protection |
| Compressor thermal overload | Built into windings | Over-temperature |
| MOV surge suppressor | 230V AC line | Transient protection |
| EMI filter | On AC input | CE EMC compliance |

### 2.3 Internal Wiring

| Connection | Cable Type | Size | Notes |
|------------|-----------|------|-------|
| AC input → inverter | H07RN-F flexible rubber | 2.5 mm² (16A) / 6 mm² (32A) | — |
| Inverter → compressor | Shielded 3-core | 1.5 mm² | Short run <1 m |
| 24V bus → fans | PVC | 1.5 mm² | With inline fuse |
| Controller → EEV | 4-core stepper cable | 0.5 mm² | Shielded |
| Sensor wires | Shielded pair | 0.5 mm² | Star grounded at controller |
| CAN bus (internal) | Twisted pair shielded | 0.75 mm² | To M12 connector |

---

## 3. Refrigeration Specification

### 3.1 Refrigerant

| Parameter | Value |
|-----------|-------|
| Refrigerant (units ≥5 kW) | **R-32** |
| Refrigerant (units ≤3 kW) | R-290 (propane), where charge limit permits |
| GWP | 675 (R-32) / 3 (R-290) |
| Flammability class | A2L (R-32) / A3 (R-290) |
| COP at rated conditions | 3.0–3.5 |
| Oil type | POE (Polyolester) |

### 3.2 Compressor

| Parameter | 150 kW (9 kW unit) | 350 kW (20 kW unit) |
|-----------|---------------------|---------------------|
| Type | Rotary or scroll, DC inverter-driven | Scroll, DC inverter-driven |
| Cooling capacity | 9 kW at 35°C ambient | 20 kW at 35°C ambient |
| Electrical input | 2–3 kW | 5–7 kW |
| Power supply | 400V AC single-phase (L-L) | 400V AC single-phase (L-L) |
| Speed range | 1800–7200 RPM | 1800–7200 RPM |
| Start method | Soft-start via inverter (no inrush) | Soft-start via inverter |
| Sound level | <55 dB(A) at 1 m | <60 dB(A) at 1 m |
| Vibration isolation | Rubber grommets, spring mounts | Rubber grommets, spring mounts |

### 3.3 Electronic Expansion Valve (EEV)

| Parameter | Value |
|-----------|-------|
| Type | Electronic Expansion Valve |
| Actuation | Stepper motor, driven by HVAC controller |
| Superheat target | 5–8°C |
| Response time | <2 s full stroke |

### 3.4 Heat Exchangers

**Evaporator Coil (Internal — Cabinet Side)**

| Parameter | 150 kW | 350 kW |
|-----------|--------|--------|
| Type | Fin-and-tube, copper tube / aluminum fin | Fin-and-tube |
| Face area | 300×200 mm | 400×300 mm |
| Fin pitch | 2.0–2.5 mm | 2.0–2.5 mm |
| Rows | 3–4 | 3–4 |
| Coating | Epoxy-coated fins | Epoxy-coated fins |
| Condensate drain | Drip tray with drain hose to exterior | Drip tray with drain hose |
| Defrost | Hot-gas bypass or reverse cycle | Hot-gas bypass or reverse cycle |

**Condenser Coil (External — Ambient Side)**

| Parameter | Value |
|-----------|-------|
| Type | Fin-and-tube or microchannel |
| Face area | 400×300 mm (150 kW) / 500×400 mm (350 kW) |
| Fin pitch | 1.5–2.0 mm |
| Design ambient | Up to 50°C |
| Coating | Hydrophilic |
| Cleaning access | Removable guard for pressure washing |

---

## 4. Airflow Specification

### 4.1 Internal Circulation Fan (Evaporator Blower)

| Parameter | 150 kW | 350 kW |
|-----------|--------|--------|
| Type | Centrifugal blower, brushless DC external rotor | Same |
| Voltage | 24V DC | 24V DC |
| Airflow | 500–1500 CFM | 1500–3500 CFM |
| Static pressure | 100–300 Pa | 100–300 Pa |
| Speed control | PWM (0–100%) | PWM (0–100%) |
| Power consumption | 50–150W | 150–400W |
| Feedback | Tachometer pulse | Tachometer pulse |
| Noise | <50 dB(A) at 1 m | <55 dB(A) at 1 m |
| Bearing life | >60,000 hr (L10) | >60,000 hr (L10) |

### 4.2 External Condenser Fan

| Parameter | Value |
|-----------|-------|
| Type | Axial fan, brushless DC external rotor |
| Voltage | 24V DC |
| Airflow | 800–2000 CFM |
| Speed control | PWM, modulated on condenser pressure/temperature |
| Power consumption | 50–200W |
| Feedback | Tachometer pulse |
| Weather protection | Rain guard / drip shield |
| Bearing life | >60,000 hr |

### 4.3 Airflow Path

Hot cabinet air (45–55°C) exits via the hot air port, passes over the evaporator coil, and returns as cooled air (5–15°C below setpoint) through the cold air port. Inside the cabinet, a plenum with adjustable baffles directs 40% airflow to each power module zone and 20% to control electronics. See [[docs/Hardware/06 - HVAC Clip-On Unit Hardware Design|06 - HVAC Clip-On Unit Hardware Design]] §5 for detailed airflow diagrams.

---

## 5. Thermal Performance

| Parameter | 150 kW (9 kW unit) | 350 kW (20 kW unit) |
|-----------|---------------------|---------------------|
| Cooling capacity (thermal) | 9 kW | 20 kW |
| Electrical input | ~3 kW | ~7 kW |
| COP at 35°C ambient | ~3.0 | ~2.8 |
| Target cabinet temperature | 30–35°C during charging | 30–35°C during charging |
| Operating ambient range | -30°C to +50°C (with cold climate package) | -30°C to +50°C |
| Condenser design ambient | 50°C max | 50°C max |
| Thermal derating coordination | Derate charger at cabinet >40°C; shutdown at >50°C | Same |

### 5.1 Thermal Derating Thresholds

| Cabinet Temp | HVAC Action | Charger Action |
|--------------|-------------|----------------|
| ≤30°C | Normal operation | Full power |
| 30–40°C | COOLING mode, full compressor | Full power |
| 40–45°C | Sets `derate_request` flag | Reduce output by 25% |
| 45–50°C | Sends fault 0x05 (critical) | Reduce output by 50% |
| >50°C | Sets `critical_fault` flag | Orderly shutdown |

---

## 6. Sensors and Safety

### 6.1 Sensor Inputs

| Sensor | Type | Range | Location |
|--------|------|-------|----------|
| T1: Cabinet air temp | NTC 10kΩ or PT1000 | -40 to +125°C | Hot air return duct |
| T2: Condenser coil temp | NTC 10kΩ | -40 to +125°C | Condenser outlet pipe |
| T3: Evaporator coil temp | NTC 10kΩ | -40 to +125°C | Evaporator outlet pipe |
| T4: Ambient temp | NTC 10kΩ | -40 to +85°C | External, shaded |
| P1: High-side pressure | Pressure transducer | 0–40 bar | Compressor discharge |
| CS1: Compressor current | Hall effect / shunt | 0–15A | Compressor supply line |

### 6.2 Safety Switches

| Switch | Type | Trip Point | Reset |
|--------|------|------------|-------|
| HP: High-pressure switch | Normally closed | Opens at 28 bar | Manual reset |
| LP: Low-pressure switch | Normally closed | Opens at 1.5 bar | Auto-reset |

### 6.3 Safety Behaviour

- HP switch open → compressor disabled, fault 0x03
- LP switch open → compressor disabled, fault 0x04
- Compressor over-current (CS1 > 110%) → compressor disabled, fault 0x01
- Condenser over-temperature (T2 > 65°C) → compressor disabled, fan derate
- Phytec SBC heartbeat lost >6 s → autonomous safe mode (last setpoint, compressor capped at 70%)

---

## 7. Control Interface

### 7.1 HVAC Controller MCU

| Parameter | Value |
|-----------|-------|
| MCU | STM32F103 or RP2350 |
| Architecture | Bare-metal or FreeRTOS |
| CAN interface | CAN 2.0B, 250 kbps (to Phytec SBC via CAN #3) |
| ADC channels | 6 (4× temperature, 1× pressure, 1× current) |
| PWM outputs | 3 (compressor inverter, internal fan, external fan) |
| Digital outputs | 3 (EEV stepper, heater relay, fault LED) |
| Digital inputs | 2 (HP switch, LP switch) |
| Power supply | 24V DC input → internal 3.3V/5V regulator |
| Firmware update | CAN bootloader or UART/SWD debug port |
| Watchdog | Internal IWDG; autonomous safe mode on Phytec SBC timeout |

### 7.2 CAN Bus Interface

The HVAC communicates with the Phytec SBC Main Controller over CAN #3 (dedicated, isolated). The full message dictionary, signal definitions, and EVerest integration are defined in [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]].

| Parameter | Value |
|-----------|-------|
| Bus | CAN #3 (dedicated HVAC) |
| Bitrate | 250 kbps |
| Protocol | CAN 2.0B (29-bit identifiers) |
| Connector | M12 4-pin, A-coded, IP67 mated |
| Isolation | 3 kV galvanic (at Phytec SBC side) |
| Termination | 120 Ω at each end |
| Cable | Shielded twisted pair, max 5 m |

### 7.3 Operating Modes

| Mode | Description |
|------|-------------|
| OFF | All outputs disabled |
| STANDBY | Fans at minimum (10%), compressor off, monitoring only |
| COOLING | Active cooling — PID on cabinet temp setpoint (default 35°C) |
| HEATING | PTC heater active, fans at 50%, compressor off |
| DEFROST | Hot-gas bypass or reverse cycle, 5–10 min duration |
| VENTILATION | Fans only, compressor off (energy-saving, mild conditions) |

---

## 8. Mechanical Specification

### 8.1 Enclosure

| Parameter | 150 kW Variant | 350 kW Variant |
|-----------|----------------|----------------|
| Dimensions (H×W×D) | 600×400×300 mm | 800×500×400 mm |
| Weight | 25–35 kg | 45–65 kg |
| Material | Powder-coated galvanised steel | Powder-coated galvanised steel |
| IP Rating | IP55 | IP55 |
| Colour | RAL 7035 (light grey) | RAL 7035 |
| Mounting | Side, rear, or top of cabinet | Side or rear |

### 8.2 Clip-On Interface

The clip-on interface is the mechanical and electrical boundary between the HVAC unit and the cabinet. It supports tool-free attachment (aside from 4 securing bolts) and quick-disconnect of all services.

| Component | Specification |
|-----------|---------------|
| Air ports (×2) | 200×150 mm openings with EPDM foam gasket seal |
| Foam gaskets | Closed-cell EPDM, 10 mm thick, self-adhesive |
| Power connector | Industrial multipole (e.g. Harting Han), AC + DC + PE |
| CAN connector | M12 4-pin, A-coded, IP67 mated |
| Condensate drain | 10 mm barb fitting with silicone hose |
| Mounting bolts | 4× M8×25 with captive nuts on cabinet side (12 Nm, star pattern) |
| Alignment pins | 2× 8 mm dowel pins, stainless steel |
| Quick-release latches | Spring-loaded (optional, for frequent service) |

### 8.3 FRU Swap

Target swap time: **30 minutes** including functional verification. The procedure involves disconnecting power/CAN/drain, removing 4× M8 bolts, sliding the unit off alignment pins, and reversing for the replacement. See [[docs/Hardware/06 - HVAC Clip-On Unit Hardware Design|06 - HVAC Clip-On Unit Hardware Design]] §10.2 for the full FRU swap procedure.

---

## 9. Cold Climate Package (Optional)

For deployments where ambient temperature drops below -20°C.

### 9.1 Components

| Component | Specification | Purpose |
|-----------|---------------|---------|
| PTC heater element | 500W–1 kW, 400V AC | Maintains cabinet above 0°C during idle |
| Crankcase heater | 30–50W, 230V AC | Keeps compressor oil warm for reliable cold starts |
| Heated condensate drain | Self-regulating heat tape | Prevents condensate freeze-up |
| Low-ambient fan cycling | Condenser fan speed reduction | Maintains head pressure at low ambient |

### 9.2 Heating Mode

Activates when cabinet temperature (T1) drops below 5°C:
1. Compressor OFF
2. PTC heater relay ON
3. Internal blower fan at 50%
4. External condenser fan OFF
5. PID control on T1 — cycles heater at setpoint (typically 10°C idle)

Transition to cooling occurs when a charging session starts and heat load drives T1 above the cooling setpoint.

### 9.3 Defrost Cycle

When evaporator temperature (T3) is below 0°C for >10 minutes during cooling, ice buildup triggers a defrost cycle:
1. Internal fan reduced to 10%
2. Hot-gas bypass valve activated (or reverse refrigerant flow)
3. Evaporator heated until T3 > 10°C (5–10 min typical)
4. Normal cooling resumes, fan ramps back up

Defrost is reported to the Phytec SBC via CAN operating mode = DEFROST (0x101).

---

## 10. References

| Document | Location |
|----------|----------|
| HVAC Hardware Design | [[docs/Hardware/06 - HVAC Clip-On Unit Hardware Design|06 - HVAC Clip-On Unit Hardware Design]] |
| HVAC CAN Interface | [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]] |
| Component BOM | [[docs/Components/10 - HVAC Clip-On Unit|10 - HVAC Clip-On Unit]] |
| Backplane Power | [[docs/Hardware/04 - Backplane Power Management|04 - Backplane Power Management]] |
| System Architecture | [[docs/System/01 - System Architecture|01 - System Architecture]] |
| Cabinet Layout | [[docs/Hardware/03 - Cabinet Layout|03 - Cabinet Layout]] |
| DCFC Comparison | [[research/03 - 150kW DCFC Comparison|03 - 150kW DCFC Comparison]] |

**Standards**: IEC 60335-2-40 (heat pumps/AC safety), EN 378 (refrigerant systems), IEC 60079-15 (R-290 protection)

---

**Document Version**: 1.0
**Last Updated**: 2026-02-27
**Prepared by**: Mechanical & Thermal Engineering
