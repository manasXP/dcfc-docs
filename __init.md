
Goal: DC Fast Charger Design

A specification for a DC fast charger (DCFC) includes electrical, communication, and environmental requirements and adheres to key industry standards. These chargers convert three-phase AC input to a wide-range DC output to serve both 400V and 800V vehicles.



Key Specifications for an 800V DC Fast Charger

| Category                | Specification Details                                                                                                                              | Notes                                                                                                      |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Input (AC Supply)       | Three-phase 400-480 Vac nominal (EU/US grids)                                                                                                      | Requires high-capacity service and potential transformer upgrades.                                         |
| —                       | Frequency: 50/60 Hz                                                                                                                                | —                                                                                                          |
| —                       | Power Factor: &gt;0.99                                                                                                                             | Ensures efficient power draw from the grid.                                                                |
| Output (DC to EV)       | Voltage Range: 200 V to 1000 V DC                                                                                                                  | Wide range ensures compatibility with both 400V and 800V EVs.                                              |
| —                       | Max Output Power: 50 kW up to 350+ kW                                                                                                              | Higher power (350 kW) is typical for _ultra-fast_ 800V systems.                                            |
| —                       | Max Output Current: Dependent on power and voltage (e.g., 500A max at high power)                                                                  | The current (Amperage) is key to managing heat and charging speed.                                         |
| Communication           | Vehicle Communication: Isolated CAN, UART, or Ethernet                                                                                             | Adheres to standards like IEC 61851-23/24 for communication with the EV's Battery Management System (BMS). |
| —                       | Network Communication: OCPP (Open Charge Point Protocol) for network management (e.g., payment, monitoring)                                        | —                                                                                                          |
| User Interface          | LCD/Touchscreen display, physical start/stop buttons, LED indicators, and potential for payment terminal integration.                              | —                                                                                                          |
| Connectors              | CCS (Combined Charging System) Combo 1 or Combo 2 is the prevailing standard for 800V systems. CHAdeMO is an alternative standard in some regions. | Requires robust, often liquid-cooled, high-amperage cables.                                                |
| Efficiency & Components | Peak Efficiency: Typically &gt;96%                                                                                                                 | Achieved using advanced components like Silicon Carbide (SiC) MOSFETs.                                     |
| —                       | Architecture: Often uses modular power blocks (e.g., 25 kW or 50 kW modules) stacked in parallel to achieve higher total output power.             | —                                                                                                          |
| Safety & Environmental  | Protections: Over-Voltage Protection (OVP), Over-Current Protection (OCP), Short Circuit (SC), Ground Fault Detection, and thermal management.     | —                                                                                                          |
| —                       | Operating Temperature: Wide range, typically suitable for outdoor use (e.g., 0°C to 40°C or wider).                                                | Requires robust, IP-rated construction.                                                                    |
| —                       | Standards Compliance: Follows IEC 61851 (electrical safety/performance) and ISO 15118 (Plug & Charge) guidelines.                                  | —                                                                                                          |





EVerest (Electric Vehicle Energy System) is an open-source, modular software framework, often called an "operating system," for EV chargers, managed by [the Linux Foundation Energy](https://lfenergy.org/projects/everest/). It provides a common, customizable base layer for charging station manufacturers, supporting protocols like OCPP and ISO 15118, and enabling features like load balancing, energy management, and hardware integration, aiming to standardize and accelerate the development of reliable and interoperable charging infrastructure. https://github.com/EVerest/EVerest/blob/main/README.md

## 1. Key Aspects of EVerest

- Open-Source & Modular: Built on a microservices architecture (Linux processes communicating via MQTT) for flexibility, allowing interchangeable modules for different functionalities.
- Comprehensive Stack: Includes core charging logic, protocol support (OCPP, ISO 15118, IEC 61851), hardware drivers (meters, readers), energy management, and simulation tools.
- Standardization: Provides a common base for manufacturers, letting them focus on value-added features rather than basic commodity software, improving interoperability.
- Community-Driven: Developed under the Linux Foundation Energy (LF Energy), with significant contributions from various industry players.
- Wide Adoption: Used in numerous production chargers, with significant growth projected, powering both AC and DC chargers.

## 2. How it Works:

- Microservices: Individual software components (modules) run as separate processes, communicating through defined interfaces.
- MQTT: Message Queuing Telemetry Transport handles communication between these loosely coupled modules.
- Framework Manager: Manages the lifecycle of these modules, making development and configuration easier.
-
In essence, EVerest is a foundational software platform that simplifies building complex EV charging solutions, promoting faster innovation and more reliable chargers.

## Documentation

### System
1. [[docs/System/01 - System Architecture|01 - System Architecture]]
2. [[docs/System/02 - Technical Specifications|02 - Technical Specifications]]
3. [[docs/System/03 - Standards Compliance|03 - Standards Compliance]]

### Hardware
1. [[docs/Hardware/01 - Hardware Components|01 - Hardware Components]]
2. [[docs/Hardware/02 - Electric Wiring Diagram|02 - Electric Wiring Diagram]]
3. [[docs/Hardware/03 - Cabinet Layout|03 - Cabinet Layout]]
4. [[docs/Hardware/04 - Backplane Power Management|04 - Backplane Power Management]]
5. [[docs/Hardware/05 - DC Output Contactor and Pre-Charge Circuit|05 - DC Output Contactor and Pre-Charge Circuit]]
6. [[docs/Hardware/06 - HVAC Clip-On Unit Hardware Design|06 - HVAC Clip-On Unit Hardware Design]]
7. [[docs/Hardware/07 - CCS Connector and Liquid-Cooled Cable Assembly|07 - CCS Connector and Liquid-Cooled Cable Assembly]]
8. [[docs/Hardware/08 - Power Module Hardware Design|08 - Power Module Hardware Design]]

### HVAC
1. [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]]

### Software
1. [[docs/Software/01 - Software Framework|01 - Software Framework]]
2. [[docs/Software/02 - Communication Protocols|02 - Communication Protocols]]
3. [[docs/Software/03 - Safety Supervisor Controller|03 - Safety Supervisor Controller]]
4. [[docs/Software/04 - Power Module CAN Bus Interface|04 - Power Module CAN Bus Interface]]
5. [[docs/Software/05 - ISO 15118 Vehicle Communication|05 - ISO 15118 Vehicle Communication]]

### Software / EVerest
1. [[docs/Software/EVerest/01 - EVerest Safety Supervisor Integration|01 - EVerest Safety Supervisor Integration]]
2. [[docs/Software/EVerest/02 - EVerest Power Module Driver|02 - EVerest Power Module Driver]]
3. [[docs/Software/EVerest/03 - EVerest OCPP201 Backend Integration|03 - EVerest OCPP201 Backend Integration]]
4. [[docs/Software/EVerest/04 - EVerest HVAC Driver|04 - EVerest HVAC Driver]]
5. [[docs/Software/EVerest/05 - EVerest Energy Manager|05 - EVerest Energy Manager]]
6. [[docs/Software/EVerest/06 - EVerest EvseManager|06 - EVerest EvseManager]]
7. [[docs/Software/EVerest/07 - EVerest Auth Module|07 - EVerest Auth Module]]

## Diagrams

1. [[diagrams/HVAC-CANBus-Connection.excalidraw|HVAC CANBus Electrical Connection]]

## Research

1. [[research/01 - Safety Philosophy|01 - Safety Philosophy]]
2. [[research/02 - CM5 based Main Controller|02 - CM5 based Main Controller]]
3. [[research/03 - 150kW DCFC Comparison|03 - 150kW DCFC Comparison]]
4. [[research/04 - Phytec SBC Replacement|04 - Phytec SBC Replacement]]
5. [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]]
