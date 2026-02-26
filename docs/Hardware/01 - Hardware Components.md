# Hardware Components

Tags: #dcfc #hardware #components #power-electronics

Related: [[__Workspaces/ECG/__init]] | [[03 - Standards Compliance]]

## 1. Overview

This document provides detailed specifications and requirements for the key hardware components of a DC Fast Charger (DCFC) system. These components work together to safely convert AC grid power to DC power for electric vehicle battery charging.

## 2. Power Conversion Module

### 2.1 AC/DC Converter

**Function**: Converts 3-phase AC grid power to high-voltage DC output

**Key Specifications**:
- Input: 3-phase AC 400V/480V (50/60 Hz)
- Output: 200-1000V DC
- Power rating: 50-350 kW per module
- Efficiency: >95% at rated load
- Power factor: >0.99
- THD (Total Harmonic Distortion): <5%

**Architecture**:
- Vienna rectifier or three-level PFC (Power Factor Correction)
- DC-DC converter stage for voltage regulation
- Modular design for scalability and redundancy

**Components**:
- **SiC MOSFETs**: Silicon Carbide transistors for high-efficiency switching
- **Gate Drivers**: Isolated gate drive circuits for MOSFET control
- **DC Link Capacitors**: High-voltage film or electrolytic capacitors
- **Inductors/Transformers**: Magnetic components for energy storage and isolation
- **DSP/FPGA Controller**: Real-time control of power conversion

### 2.2 Power Module Ratings

| Parameter | 50 kW Module | 150 kW Module | 350 kW Module |
|-----------|--------------|---------------|---------------|
| Max Output Current | 125A | 375A | 500A |
| Voltage Range | 200-500V | 200-920V | 200-1000V |
| Cooling | Liquid | Liquid | Liquid |
| Parallel Units | Up to 4 | Up to 2 | Single |
| Dimensions (HxWxD) | 450x400x250mm | 600x500x300mm | 800x600x400mm |
| Weight | 35 kg | 80 kg | 150 kg |

## 3. Charging Connectors

### 3.1 CCS Combo Connector (Combined Charging System)

**Standard**: IEC 62196-3 / SAE J1772 Combo

**Pin Configuration**:
- 2x DC power pins (DC+ and DC-)
- 5x AC pins (for AC charging compatibility)
- 2x communication pins (CP - Control Pilot, PP - Proximity Pilot)

**Ratings**:
- Maximum current: 500A DC
- Maximum voltage: 1000V DC
- Temperature rating: -40°C to +50°C
- IP rating: IP54 (mated), IP44 (unmated)
- Mechanical durability: >10,000 mating cycles

**Cable Specifications**:
- Length: 2-3 meters (standard)
- Conductor size: 70-95 mm² for high-power applications
- Cooling: Liquid-cooled cable for >150 kW systems
- Flexibility: Highly flexible TPE or rubber jacket

### 3.2 CHAdeMO Connector (Optional)

**Standard**: CHAdeMO 2.0 / 3.0

**Ratings**:
- CHAdeMO 2.0: Up to 400A @ 1000V DC (400 kW)
- CHAdeMO 3.0: Up to 600A @ 1500V DC (900 kW)
- CAN-based communication protocol

### 3.3 NACS (North American Charging Standard)

**Standard**: SAE J3400

**Features**:
- Combined AC and DC charging
- Compact design
- Up to 1000V DC, 500A (500 kW capable)
- Integrated communication pins

## 4. Cooling System

### 4.1 Liquid Cooling System

**Purpose**: Thermal management of power modules, cables, and connectors

**Components**:
- **Coolant Pump**: Variable speed pump (12V/24V DC)
- **Heat Exchanger/Radiator**: Air-to-liquid or liquid-to-liquid
- **Coolant Reservoir**: Expansion tank with level sensor
- **Coolant**: Glycol-water mixture (50/50) or dielectric fluid
- **Temperature Sensors**: RTD or thermistor sensors
- **Fans**: High-efficiency axial fans with PWM control

**Specifications**:
- Flow rate: 10-30 L/min
- Operating temperature: 40-60°C
- Coolant pressure: 1-3 bar
- Heat dissipation: 5-10% of charging power (2.5-35 kW)

### 4.2 Air Cooling (Lower Power Systems)

**Application**: <50 kW chargers

**Components**:
- Forced air fans
- Heat sinks on power semiconductors
- Dust filters (replaceable)
- Ambient temperature monitoring

## 5. Protection and Safety Devices

### 5.1 Electrical Protection

**Overcurrent Protection**:
- DC circuit breakers or contactors
- Electronic fuses (semiconductor-based)
- Current sensors (Hall effect or Rogowski coil)
- Trip rating: 110-125% of maximum rated current

**Overvoltage/Undervoltage Protection**:
- Voltage monitoring relays
- Surge protection devices (SPD)
- MOV (Metal Oxide Varistor) suppressors
- Voltage limits: 150-1000V DC (programmable)

**Ground Fault Protection**:
- Residual current detection (Type B RCD)
- Insulation monitoring device (IMD)
- Ground continuity verification
- Trip threshold: 20-30 mA DC

**Arc Fault Detection**:
- High-frequency current sensors
- Arc fault circuit interrupter (AFCI)
- Response time: <100 ms

### 5.2 Mechanical Safety

**Emergency Stop**:
- Red mushroom-head button (front panel accessible)
- Hardwired safety circuit (no software dependency)
- Immediately de-energizes output

**Door Interlock**:
- Magnetic or mechanical door switches
- Prevents access during operation
- Disables high voltage when opened

**Cable Management**:
- Cable retractor or holster
- Strain relief at connector
- Anti-kink design

## 6. Metering and Measurement

### 6.1 Energy Meter

**Purpose**: Accurate billing and energy accounting

**Requirements**:
- MID (Measuring Instruments Directive) certified
- Accuracy class: 0.5 or better
- Measurements: kWh, voltage, current, power factor
- Communication: Modbus RTU/TCP, Ethernet

### 6.2 Current Sensors

**Types**:
- Hall effect current sensors
- Rogowski coils (for high current)
- Shunt resistors (calibration reference)

**Specifications**:
- Range: 0-600A
- Accuracy: ±0.5%
- Bandwidth: DC to 100 kHz

### 6.3 Voltage Sensors

**Types**:
- Resistive divider with isolation amplifier
- Differential voltage measurement

**Specifications**:
- Range: 0-1200V
- Isolation: >4 kV
- Accuracy: ±0.5%

## 7. Communication Hardware

### 7.1 Vehicle Communication

**ISO 15118 (PLC - Power Line Communication)**:
- SLAC modem (Signal Level Attenuation Characterization)
- HomePlug Green PHY compliant
- Communication over CP line

**CAN Bus Interface**:
- CAN transceiver (ISO 11898)
- 250 kbps or 500 kbps
- Twisted pair with 120Ω termination

### 7.2 Network Communication

**Ethernet Interface**:
- 100BASE-TX or 1000BASE-T
- RJ45 connector with magnetics
- PoE support (optional)

**4G/5G Cellular Modem**:
- LTE Cat-4 or higher
- External antenna with SMA connector
- SIM card slot (optional eSIM)

**Wi-Fi Module** (optional):
- 2.4/5 GHz dual-band
- 802.11ac or 802.11ax
- External or internal antenna

### 7.3 OCPP Communication

**Connection Method**:
- WebSocket over TLS
- MQTT (optional, for cloud integration)
- Requires stable internet connection

## 8. User Interface Hardware

### 8.1 Display

**Type**: Industrial touchscreen LCD

**Specifications**:
- Size: 7-10 inch diagonal
- Resolution: 800x480 or 1280x800
- Brightness: 500-1000 cd/m²
- Operating temp: -20°C to +70°C
- IP rating: IP65 (front panel)
- Sunlight readable: Yes

### 8.2 RFID/NFC Reader

**Purpose**: User authentication and payment

**Standards**:
- ISO 14443 (NFC, MIFARE)
- ISO 15693 (vicinity cards)
- EMV contactless payment

**Features**:
- Read range: 5-10 cm
- Multi-protocol support
- LED status indicators

### 8.3 Payment Terminal (Optional)

**Features**:
- EMV chip and PIN
- Contactless payment (NFC)
- PCI DSS compliant
- Receipt printer

### 8.4 LED Indicators

**Status LEDs**:
- Power on (green)
- Charging (blue/pulsing)
- Error (red)
- Available (green/idle)
- Network connected (green)

**Location**: Visible from vehicle during charging

## 9. Enclosure and Mechanical

### 9.1 Cabinet Design

**Materials**:
- Powder-coated steel or stainless steel
- UV-resistant polycarbonate (windows/display)
- Aluminum heat sinks

**Protection Rating**:
- IP54 minimum (outdoor installation)
- IK10 impact resistance (front panel)
- NEMA 3R or 4X (North America)

**Ventilation**:
- Forced air circulation
- Dust filters
- Temperature-controlled fans
- Condensation drainage

### 9.2 Mounting

**Floor Mounting**:
- Anchor bolts (M12 or larger)
- Anti-vibration pads
- Cable entry from bottom or rear

**Pedestal Design**:
- Freestanding column (typical)
- Height: 1400-1600 mm
- Footprint: 600x400 mm to 800x600 mm

## 10. Auxiliary Systems

### 10.1 Internal Power Supply

**Purpose**: Power control electronics, display, communication modules

**Specifications**:
- Input: AC grid voltage
- Outputs: 24V DC, 12V DC, 5V DC
- Power rating: 100-300W
- Backup battery (optional): For graceful shutdown

### 10.2 Battery Backup (UPS)

**Purpose**: Complete ongoing transaction during grid failure

**Capacity**: 5-15 minutes at full power (for completion of session)

**Type**: Lithium-ion or supercapacitor

### 10.3 Environmental Sensors

**Sensors**:
- Ambient temperature (thermistor)
- Internal cabinet temperature
- Humidity sensor (capacitive)
- Smoke detector (optional)

**Monitoring**:
- Continuous logging
- Alerts on out-of-range conditions
- Integration with OCPP diagnostics

## 11. Bill of Materials (BOM) Categories

### 11.1 Major Components
1. Power conversion modules
2. Charging cables and connectors
3. Cooling system (pump, radiator, fans)
4. Enclosure and mounting hardware
5. User interface (display, RFID reader)

### 11.2 Control and Communication
1. Main controller (embedded computer or PLC)
2. Communication modules (Ethernet, cellular)
3. ISO 15118 modem
4. Energy meter

### 11.3 Protection and Safety
1. Circuit breakers and contactors
2. Fuses and surge protectors
3. Emergency stop button
4. Ground fault monitoring

### 11.4 Wiring and Connectors
1. Power cables (AC input, DC output)
2. Control wiring and cable harnesses
3. Terminal blocks and connectors
4. Cable glands and strain reliefs

## 12. Maintenance Considerations

### 12.1 Serviceable Components

**Regular Replacement**:
- Air filters (quarterly)
- Coolant (annually)
- Connector wear inspection (quarterly)
- Cable inspection (monthly)

**Predictive Maintenance**:
- Capacitor health monitoring
- Fan bearing wear
- Contactor contact wear
- Thermal paste degradation

### 12.2 Spare Parts Inventory

**Critical Spares**:
- Power module (or repair kit)
- Charging cable assembly
- Display/touchscreen
- Communication module
- Coolant pump

## 13. References

- IEC 62196: Plugs, socket-outlets, vehicle connectors
- IEC 61851: EV conductive charging system
- [[03 - Standards Compliance]] - Related regulatory requirements
- [[__Workspaces/ECG/__init]] - System architecture and specifications

## 14. Related Documentation

- Power conversion design
- Thermal management calculations
- Safety circuit diagrams
- Component datasheets
