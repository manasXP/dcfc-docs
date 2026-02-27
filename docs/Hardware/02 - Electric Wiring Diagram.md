# Electric Wiring Diagram - DC Fast Charger

Tags: #dcfc #wiring #electrical #schematic #diagram

Related: [[__Workspaces/ECG/__init]] | [[01 - Hardware Components]] | [[03 - Standards Compliance]]

## 1. Overview

This document provides a comprehensive electrical wiring diagram for a DC Fast Charger system, including power distribution, control circuits, safety systems, and communication interfaces.

## 2. System Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                        DC FAST CHARGER SYSTEM                          │
│                                                                        │
│  ┌─────────────┐      ┌──────────────┐      ┌─────────────┐            │
│  │   3-Phase   │      │    AC/DC     │      │  DC Output  │            │
│  │  AC Input   │─────▶│   Power      │─────▶│  Circuit    │──┐         │
│  │  400/480V   │      │  Conversion  │      │  200-1000V  │  │         │
│  │             │      │   Module     │      │             │  │         │
│  └─────────────┘      └──────────────┘      └─────────────┘  │         │
│         │                     │                   │          │         │
│         │              ┌──────┴──────┐            │          │         │
│         │              │   Cooling   │            │          │         │
│         │              │   System    │            │          │         │
│         │              └─────────────┘            │          │         │
│         │                                         │          │         │
│  ┌──────┴───────┐      ┌──────────────┐      ┌────┴────┐     │         │
│  │  Protection  │      │   Control    │      │  Safety │     │         │
│  │   Devices    │◀────▶│  Controller  │◀────▶│ Circuit │     │         │
│  │              │      │   (PLC/CPU)  │      │         │     │         │
│  └──────────────┘      └──────┬───────┘      └─────────┘     │         │
│                               │                              │         │
│                        ┌──────┴───────┐                      │         │
│                        │ User Interface│                     │         │
│                        │   Display     │                     │         │
│                        │   RFID/NFC    │                     │         │
│                        └───────────────┘                     │         │
│                                                              │         │
│  ┌─────────────┐      ┌──────────────┐                       │         │
│  │ Network/    │      │   Energy     │                       │         │
│  │ Comms       │◀────▶│   Meter      │◀──────────────────────┘         │
│  │ (OCPP)      │      │   (MID)      │                                 │
│  └─────────────┘      └──────────────┘                                 │
│                                                                        │
│                        ┌──────────────┐                                │
│                        │   Vehicle    │                                │
│                        │ Connector    │──────────────────────▶ TO EV   │
│                        │ (CCS/NACS)   │                                │
│                        └──────────────┘                                │
└────────────────────────────────────────────────────────────────────────┘
```

## 3. Main Power Circuit

### 3.1 AC Input Circuit

```
GRID 3-Phase AC (L1, L2, L3, PE)
│
├─── Main Disconnect Switch (400A, 3-pole)
│    │
│    ├─── Surge Protection Device (SPD Type 2)
│    │
│    └─── AC Circuit Breaker (350A, 3-pole, Type C)
│         │
│         ├─── Current Transformers (CT1, CT2, CT3) - Metering
│         │
│         ├─── AC Contactor (K1) - Main input contactor
│         │    Coil: 24V DC
│         │    Rating: 400A, 690V AC
│         │
│         └─── EMI Filter (3-phase)
│              │
│              └─── To AC/DC Power Modules
```

**Wire Specifications**:
- L1, L2, L3: 120 mm² Cu, XLPE insulated, 90°C rated
- PE (Protective Earth): 95 mm² Cu, green/yellow
- Length: Keep < 5m to minimize voltage drop

### 3.2 DC Output Circuit

```
AC/DC Power Module Output (DC+ and DC-)
│
├─── DC Link Capacitor Bank (2000 µF, 1200V)
│
├─── DC Circuit Breaker (CB1) - 500A, 1000V DC
│
├─── Output Contactor (K2) - Positive rail
│    Coil: 24V DC
│    Rating: 500A, 1000V DC
│    Auxiliary contacts: 2 NO + 2 NC
│
├─── Output Contactor (K3) - Negative rail
│    Rating: 500A, 1000V DC
│
├─── DC Fuses (F1, F2) - 500A gR semiconductor fuses
│
├─── Current Sensor (Hall Effect) - 600A range
│    Output: 4-20 mA or 0-10V
│
├─── Voltage Sensor (Isolated divider)
│    Range: 0-1200V
│    Output: 0-10V
│
├─── Pre-charge Resistor Circuit
│    R1: 100Ω, 100W (with bypass contactor K4)
│    Purpose: Soft-start for capacitor charging
│
└─── To Charging Cable/Connector
     │
     ├─── DC+ (70-95 mm² Cu, red, liquid-cooled for >150kW)
     └─── DC- (70-95 mm² Cu, black, liquid-cooled for >150kW)
```

## 4. Control and Safety Circuit (24V DC System)

### 4.1 24V DC Power Supply

```
AC Input (3-phase L-L)
│
└─── Switched-Mode Power Supply (SMPS)
     Input: 400V AC (L-L), 3-phase rated
     Output: 24V DC, 10A (240W)
     │
     ├─── 24V DC Bus (positive)
     │    │
     │    ├─── Fuse F10 (10A)
     │    │
     │    └─── Filter capacitor (1000 µF)
     │
     └─── 0V DC (common/ground)
```

### 4.2 Safety Interlock Circuit

```
24V DC Bus
│
└─── Emergency Stop Circuit (Hardwired)
     │
     ├─── E-Stop Button (NC contact) - Front panel
     │    │
     │    ├─── Door Interlock Switch (NC) - Cabinet door
     │    │
     │    ├─── Thermal Overload Switch (NC) - Power module
     │    │
     │    ├─── Safety Relay (K10) - Dual channel safety relay
     │    │    Coil: 24V DC
     │    │    Contacts: 2 NO (safety rated)
     │    │    │
     │    │    ├── Contact 1 ──▶ To Main Contactor K1 coil
     │    │    └── Contact 2 ──▶ To Output Contactors K2, K3 coils
     │    │
     │    └─── To Controller (E-Stop Status Input)
     │
     └─── Ground Fault Monitor (IMD)
          │
          ├─── Insulation Monitoring Device
          │    Range: 0-1000V DC
          │    Alarm threshold: < 100Ω/V
          │    Output: Relay contact (NC)
          │
          └─── To Safety Relay K10 (series with E-Stop)
```

### 4.3 Contactor Control Circuit

```
Controller Output (PLC/MCU)
│
├─── K1 Coil Control (AC Main Contactor)
│    │
│    ├─── 24V DC from Safety Relay K10
│    ├─── Transistor/Relay Driver (from controller DO1)
│    └─── Flyback Diode (D1)
│
├─── K2 Coil Control (DC+ Output Contactor)
│    │
│    ├─── 24V DC from Safety Relay K10
│    ├─── Transistor/Relay Driver (from controller DO2)
│    └─── Flyback Diode (D2)
│
├─── K3 Coil Control (DC- Output Contactor)
│    │
│    ├─── 24V DC from Safety Relay K10
│    ├─── Transistor/Relay Driver (from controller DO3)
│    └─── Flyback Diode (D3)
│
└─── K4 Coil Control (Pre-charge Bypass Contactor)
     │
     ├─── 24V DC
     ├─── Transistor/Relay Driver (from controller DO4)
     └─── Flyback Diode (D4)
```

## 5. Vehicle Communication Circuit

### 5.1 ISO 15118 / CCS Communication

```
Charging Connector Pins
│
├─── CP (Control Pilot) - Pin 3
│    │
│    ├─── PWM Signal Generator (1 kHz, ±12V)
│    │    From: Controller PWM output
│    │    Circuit: PWM driver + current source (1.3A/2.7A)
│    │
│    ├─── CP Voltage Monitor (ADC input to controller)
│    │    Range: -12V to +12V
│    │    Protection: Zener diodes (18V)
│    │
│    └─── PLC Modem (HomePlug Green PHY)
│         │
│         ├─── Coupling Circuit (transformer + capacitor)
│         ├─── TX/RX to ISO 15118 Controller
│         └─── To Ethernet (for higher-layer communication)
│
├─── PP (Proximity Pilot) - Pin 4
│    │
│    ├─── Resistor divider circuit
│    │    Purpose: Cable detection and current rating
│    │    R2 in cable plug (150Ω / 480Ω / 1.5kΩ / 680Ω)
│    │
│    └─── Voltage Monitor (ADC input to controller)
│         Range: 0-5V
│
└─── PE (Protective Earth) - Connected to chassis ground
```

### 5.2 CAN Bus Interface (for CHAdeMO or diagnostics)

```
Controller CAN Transceiver
│
├─── CANH (CAN High)
│    │
│    ├─── 120Ω Termination Resistor (if end node)
│    └─── To Connector Pin / External CAN bus
│
└─── CANL (CAN Low)
     │
     ├─── 120Ω Termination Resistor (if end node)
     └─── To Connector Pin / External CAN bus

Wire: Twisted pair, shielded
Specification: CAN-bus cable, 120Ω characteristic impedance
```

## 6. Cooling System Circuit

### 6.1 Liquid Cooling Control

```
24V DC Supply
│
├─── Coolant Pump
│    │
│    ├─── Type: Brushless DC pump
│    ├─── Voltage: 24V DC
│    ├─── Current: 3-5A
│    ├─── PWM Control: From controller (DO5)
│    │    Frequency: 25 kHz
│    │    Duty cycle: 0-100% (speed control)
│    │
│    └─── Tachometer Output: To controller (DI1)
│         Pulses per revolution: 2-4
│
├─── Radiator Fans (x2)
│    │
│    ├─── Type: Axial fans, brushless
│    ├─── Voltage: 24V DC
│    ├─── Current: 2A each
│    ├─── Control: PWM from controller (DO6)
│    │
│    └─── Parallel connection with current limiting
│
└─── Temperature Sensors
     │
     ├─── T1: Power module temperature (NTC 10kΩ)
     │    Connection: To controller AI1
     │
     ├─── T2: Coolant inlet temperature (NTC 10kΩ)
     │    Connection: To controller AI2
     │
     ├─── T3: Coolant outlet temperature (NTC 10kΩ)
     │    Connection: To controller AI3
     │
     └─── T4: Cable temperature (NTC 10kΩ or PT1000)
          Connection: To controller AI4
          Location: Near connector (liquid-cooled cable)
```

## 7. Metering and Measurement Circuit

### 7.1 Energy Meter Connection

```
AC Input (before conversion)
│
├─── Current Inputs
│    │
│    ├─── L1: From CT1 (100A/5A or 400A/5A)
│    ├─── L2: From CT2
│    └─── L3: From CT3
│
├─── Voltage Inputs (2-element Aron connection, 3-wire mode)
│    │
│    ├─── L1-L2: Direct connection (500V rated)
│    └─── L2-L3: Direct connection
│
└─── Communication Interface
     │
     ├─── Type: Modbus RTU (RS-485)
     │    │
     │    ├─── A+ (RS-485 positive)
     │    └─── B- (RS-485 negative)
     │         Wire: Twisted pair, shielded, 120Ω terminated
     │
     └─── To Controller RS-485 Port
          Baud rate: 9600 / 19200
          Protocol: Modbus RTU
```

### 7.2 DC Measurement Circuit

```
DC Output
│
├─── Voltage Measurement
│    │
│    ├─── Resistive Divider (1000:1)
│    │    R1: 1MΩ (high-voltage side)
│    │    R2: 1kΩ (ground side)
│    │
│    ├─── Isolation Amplifier (for safety)
│    │    Input: 0-1200V
│    │    Output: 0-10V
│    │    Isolation: 4kV
│    │
│    └─── To Controller (AI5 - Analog Input)
│         Range: 0-10V = 0-1000V DC
│
└─── Current Measurement
     │
     ├─── Hall Effect Sensor (closed-loop)
     │    Range: ±600A
     │    Output: 4-20 mA or ±10V
     │    Accuracy: ±0.5%
     │
     ├─── Signal Conditioning
     │    │
     │    └─── To Controller (AI6 - Analog Input)
     │         Range: 4-20 mA = 0-600A
     │
     └─── Shunt Resistor (backup/calibration)
          Value: 100 µΩ, 500A rated
          Voltage drop: 50 mV at 500A
          Connection: Differential amplifier to controller AI7
```

## 8. Network and Communication

### 8.1 Ethernet Connection

```
Controller Ethernet Port (RJ45)
│
├─── Magnetics Transformer (isolation)
│
├─── TX+ / TX- (Transmit pair)
├─── RX+ / RX- (Receive pair)
│
└─── To Network Switch or Router
     │
     ├─── Local Network (for OCPP backend)
     │    Protocol: TCP/IP, WebSocket over TLS
     │    Port: 443 (HTTPS) or custom
     │
     └─── Internet Connection
          Type: Fiber, DSL, or cellular gateway
```

### 8.2 4G/LTE Cellular Modem

```
Cellular Modem Module
│
├─── Power Supply
│    Input: 5V DC, 2A (from buck converter)
│    Source: 24V DC bus
│
├─── SIM Card Slot
│    Type: Mini-SIM or Micro-SIM
│    Voltage: 1.8V / 3V
│
├─── Antenna Connection
│    Type: SMA connector
│    Cable: RG174 or similar (50Ω)
│    Location: External antenna on top of enclosure
│
├─── UART or USB Interface
│    │
│    ├─── TX, RX, GND (UART)
│    │    Baud: 115200
│    │    Level: 3.3V TTL
│    │
│    └─── To Controller Communication Port
│
└─── Status LEDs
     Network registered: Green LED
     Data transmission: Blue LED (blinking)
```

## 9. User Interface Circuit

### 9.1 Touchscreen Display

```
Display Module (7-10 inch industrial LCD)
│
├─── Power Supply
│    Input: 12V DC or 24V DC, 1-2A
│    Source: From 24V DC bus (or dedicated 12V supply)
│
├─── Video Interface
│    │
│    ├─── Type: LVDS, HDMI, or RGB
│    └─── To Controller Video Output
│
├─── Touch Interface
│    │
│    ├─── Type: Capacitive or resistive
│    ├─── Protocol: I2C or USB
│    └─── To Controller I2C/USB port
│
└─── Backlight Control
     Input: PWM signal from controller
     Purpose: Brightness adjustment
```

### 9.2 RFID/NFC Reader

```
RFID Reader Module
│
├─── Power Supply
│    Input: 12V DC, 200 mA
│    Source: 24V to 12V buck converter
│
├─── Antenna
│    Type: Integrated or external coil
│    Frequency: 13.56 MHz (NFC/MIFARE)
│
├─── Communication Interface
│    │
│    ├─── Type: RS-232, Wiegand, or USB
│    │
│    ├─── Wiegand Interface (typical)
│    │    D0 (Data 0): Pulse signal
│    │    D1 (Data 1): Pulse signal
│    │    To controller DI2, DI3
│    │
│    └─── LED/Buzzer Control
│         Green LED: Success (DO7)
│         Red LED: Error (DO8)
│         Buzzer: Beep confirmation (DO9)
│
└─── Mounting
     Location: Front panel, near display
     Height: 1.2-1.4m from ground
```

### 9.3 Status LEDs

```
LED Indicator Panel
│
├─── Power LED (Green)
│    24V DC ──┬── Resistor (1kΩ) ──┬── LED ──┬── GND
│
├─── Charging LED (Blue)
│    Controller DO10 ──┬── Resistor (1kΩ) ──┬── LED ──┬── GND
│
├─── Error LED (Red)
│    Controller DO11 ──┬── Resistor (1kΩ) ──┬── LED ──┬── GND
│
└─── Network LED (Green)
     Controller DO12 ──┬── Resistor (1kΩ) ──┬── LED ──┬── GND

LED Specifications:
- Forward voltage: 2.0-3.3V
- Forward current: 20 mA
- Resistor calculation: R = (24V - Vf) / 0.02A
```

## 10. Grounding and Bonding

### 10.1 Grounding Scheme

```
Main Ground Bar (in cabinet)
│
├─── Protective Earth (PE) from Grid
│    Wire: 95 mm² Cu, green/yellow
│    Connection: Bolted connection, star washer
│
├─── Cabinet/Enclosure Ground
│    All metal parts bonded to ground bar
│    Wire: 16 mm² Cu minimum
│
├─── Power Module Chassis Ground
│    Low-impedance connection (< 0.1Ω)
│    Wire: 25 mm² Cu
│
├─── DC- (Negative) Grounding Point
│    Purpose: Optional grounding of DC- rail
│    Note: May be floating depending on design
│
├─── Signal Ground / 0V DC
│    Separate from chassis ground at single point
│    Connection: Star ground configuration
│
└─── Vehicle Connector PE
     Direct connection to ground bar
     Wire: 16 mm² Cu (integrated in cable)
     Verified before charging begins
```

## 11. Wire Color Coding

### 11.1 AC Power (IEC 60446)
- **L1**: Brown
- **L2**: Black
- **L3**: Gray
- **PE (Protective Earth)**: Green/Yellow

### 11.2 DC Power
- **DC+**: Red
- **DC-**: Black or Blue

### 11.3 Control Circuits (24V DC)
- **24V DC+**: Red
- **0V DC / Common**: Black
- **Digital Inputs**: White or Gray
- **Digital Outputs**: Yellow or Orange

### 11.4 Communication
- **CAN High**: White/Orange
- **CAN Low**: Orange/White
- **RS-485 A+**: Green
- **RS-485 B-**: Green/White
- **Ethernet**: Standard T568B wiring

### 11.5 Temperature Sensors
- **Sensor +**: Red
- **Sensor -**: Black
- **Shield**: Drain wire to ground

## 12. Terminal Block Layout

### 12.1 TB1 - AC Input Terminal Block
```
Terminal | Connection        | Wire Size
---------|-------------------|----------
1        | L1 (Phase 1)      | 120 mm²
2        | L2 (Phase 2)      | 120 mm²
3        | L3 (Phase 3)      | 120 mm²
4        | PE (Earth)        | 95 mm²
```

### 12.2 TB2 - DC Output Terminal Block
```
Terminal | Connection        | Wire Size
---------|-------------------|----------
1        | DC+ Output        | 95 mm²
2        | DC- Output        | 95 mm²
3        | PE (to connector) | 16 mm²
```

### 12.3 TB3 - Control Signals
```
Terminal | Connection              | Wire Size
---------|------------------------|----------
1        | 24V DC+                | 1.5 mm²
2        | 0V DC (Common)         | 1.5 mm²
3        | E-Stop Input (NC)      | 1.0 mm²
4        | Door Switch (NC)       | 1.0 mm²
5        | Safety Relay Output 1  | 1.0 mm²
6        | Safety Relay Output 2  | 1.0 mm²
7-10     | Spare                  | -
```

### 12.4 TB4 - Communication
```
Terminal | Connection              | Wire Size
---------|------------------------|----------
1        | CAN H                  | 0.75 mm²
2        | CAN L                  | 0.75 mm²
3        | RS-485 A+              | 0.75 mm²
4        | RS-485 B-              | 0.75 mm²
5        | Shield/Ground          | 0.75 mm²
6-8      | Spare                  | -
```

## 13. Cable Specifications Summary

### 13.1 Power Cables

| Circuit | Cable Type | Conductor Size | Insulation | Temperature Rating |
|---------|------------|----------------|------------|--------------------|
| AC Input (3-phase) | XLPE | 120 mm² | 0.6/1 kV | 90°C |
| DC Output | EPR or XLPE | 95 mm² | 1.5 kV DC | 90°C |
| DC Output (liquid-cooled) | Special flex | 70-95 mm² | 1.5 kV DC | 105°C |
| Protective Earth | PVC | 95 mm² | 450/750 V | 70°C |

### 13.2 Control Cables

| Circuit | Cable Type | Conductor Size | Shielding | Notes |
|---------|------------|----------------|-----------|-------|
| 24V DC Power | PVC multi-core | 1.5 mm² | No | Twisted pair |
| Digital I/O | Screened cable | 0.75 mm² | Yes | Drain wire |
| Analog Signals | Screened cable | 0.5 mm² | Yes | Star grounding |
| Temperature Sensors | Screened pair | 0.5 mm² | Yes | Twisted, shielded |

### 13.3 Communication Cables

| Circuit | Cable Type | Impedance | Shielding | Notes |
|---------|------------|-----------|-----------|-------|
| CAN Bus | Twisted pair | 120Ω | Yes | DeviceNet cable |
| RS-485 | Twisted pair | 120Ω | Yes | Terminated |
| Ethernet | Cat5e/Cat6 | 100Ω | Yes (STP) | Max 100m |

## 14. Protection and Circuit Breaker Ratings

### 14.1 Main Protection Devices

| Device | Type | Rating | Breaking Capacity | Curve |
|--------|------|--------|-------------------|-------|
| Main Disconnect | Load break switch | 400A | - | - |
| AC Circuit Breaker | MCB/MCCB | 350A | 50 kA | Type C |
| DC Circuit Breaker | DC-rated MCB | 500A | 10 kA (DC) | - |
| Auxiliary Supply Fuse | Glass fuse | 10A | 10 kA | Fast |

### 14.2 Contactor Specifications

| Contactor | Application | Coil Voltage | Contact Rating | Auxiliary Contacts |
|-----------|-------------|--------------|----------------|-------------------|
| K1 | AC Main | 24V DC | 400A @ 690V AC | 1 NO + 1 NC |
| K2 | DC+ Output | 24V DC | 500A @ 1000V DC | 1 NO + 1 NC |
| K3 | DC- Output | 24V DC | 500A @ 1000V DC | - |
| K4 | Pre-charge bypass | 24V DC | 50A @ 1000V DC | - |
| K10 | Safety relay | 24V DC | 6A @ 250V AC | 2 NO (safety) |

## 15. Installation Notes

### 15.1 Minimum Wire Bending Radius
- Power cables (>50 mm²): 15× cable diameter
- Control cables: 8× cable diameter
- Communication cables: 4× cable diameter

### 15.2 Torque Settings
- Power terminals (95-120 mm²): 20-25 Nm
- Medium terminals (16-35 mm²): 10-12 Nm
- Small terminals (<10 mm²): 2-3 Nm

### 15.3 Clearances and Creepage
- AC-DC isolation: ≥8 mm (400V AC)
- DC high voltage to ground: ≥10 mm (1000V DC)
- Control circuit spacing: ≥3 mm

## 16. Testing and Commissioning

### 16.1 Pre-Energization Checks
1. **Continuity Tests**
   - Protective earth continuity: <0.1Ω
   - PE from grid to connector: Verify connection
   - Control circuit continuity: All wiring

2. **Insulation Resistance**
   - AC circuit to ground: >1 MΩ @ 500V DC
   - DC circuit to ground: >1 MΩ @ 1000V DC
   - Control circuits to power: >1 MΩ @ 500V DC

3. **Polarity Checks**
   - AC phase sequence: L1-L2-L3 (clockwise)
   - DC output polarity: DC+ and DC- correctly labeled
   - Control voltage polarity: 24V DC+ and 0V

4. **Safety Circuit Function**
   - E-stop button: Breaks safety relay
   - Door interlock: Breaks safety relay
   - Safety relay: De-energizes all contactors

### 16.2 Energization Sequence
1. Apply AC power (main disconnect closed)
2. Verify 24V DC supply
3. Check safety circuit (LEDs, relay status)
4. Close AC contactor K1 (manually or via controller)
5. Initiate pre-charge sequence (K4 closed for 2-5 sec)
6. Close DC output contactors K2, K3
7. Verify DC output voltage (should be >200V)
8. Test communication (CP signal, PLC modem)
9. Perform test charge with EV or load bank

## 17. Maintenance Schedule

### 17.1 Weekly
- Visual inspection of cables and connections
- Check for loose terminals
- Verify cooling system operation

### 17.2 Monthly
- Clean air filters
- Check contactor operation (listen for arcing)
- Inspect charging cable for wear

### 17.3 Quarterly
- Insulation resistance testing
- Tighten all power terminals
- Calibrate current and voltage sensors

### 17.4 Annually
- Thermographic inspection
- Replace coolant
- Function test of all safety circuits
- Update firmware as needed

## 18. Related Documentation

- [[01 - Hardware Components]] - Component specifications
- [[03 - Standards Compliance]] - IEC 61851, ISO 15118 wiring requirements
- [[__Workspaces/ECG/__init]] - System overview and architecture

## 19. Schematic Drawing References

1. **Main Power Circuit (Sheet 1)**: AC input to DC output
2. **Control Circuit (Sheet 2)**: 24V DC control, safety interlocks
3. **Communication Circuit (Sheet 3)**: ISO 15118, CAN, Ethernet
4. **Cooling System (Sheet 4)**: Pump, fans, temperature sensors
5. **User Interface (Sheet 5)**: Display, RFID, LEDs
6. **Terminal Layout (Sheet 6)**: Terminal blocks and cable connections

---

**Document Version**: 1.0
**Last Updated**: 2026-01-08
**Prepared by**: Technical Documentation
**Approved by**: Chief Electrical Engineer
