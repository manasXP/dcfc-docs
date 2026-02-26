# DC Fast Charger Technical Specifications

## 1. Overview

This document details the technical specifications for an 800V DC Fast Charger (DCFC) design, covering electrical, communication, and environmental requirements in accordance with industry standards.

## 2. Power Specifications

### 2.1 Input (AC Supply)

| Parameter | Specification | Notes |
|-----------|---------------|-------|
| Voltage | Three-phase 400-480 Vac nominal | EU/US grids |
| Frequency | 50/60 Hz | Standard grid frequency |
| Power Factor | >0.99 | Ensures efficient power draw from the grid |
| Grid Requirements | High-capacity service | May require transformer upgrades |

### 2.2 Output (DC to EV)

| Parameter | Specification | Notes |
|-----------|---------------|-------|
| Voltage Range | 200 V to 1000 V DC | Wide range for 400V and 800V EV compatibility |
| Max Output Power | 50 kW to 350+ kW | 350 kW typical for ultra-fast 800V systems |
| Max Output Current | Up to 500A | Dependent on power and voltage; critical for heat management |
| Efficiency | >96% peak | Achieved using Silicon Carbide (SiC) MOSFETs |

### 2.3 Architecture

- **Modular Design**: Power blocks of 25 kW or 50 kW modules
- **Parallel Configuration**: Modules stacked in parallel to achieve higher total output power
- **Scalability**: Allows flexible power output configurations
- **Thermal Management**: External HVAC clip-on unit for cabinet cooling

## 3. Communication Protocols

### 3.1 Vehicle Communication

| Interface | Standard | Purpose |
|-----------|----------|---------|
| Isolated CAN | IEC 61851-23/24 | Communication with EV Battery Management System (BMS) |
| UART | IEC 61851-23/24 | Alternative vehicle interface |
| Ethernet | IEC 61851-23/24 | High-speed vehicle communication |

### 3.2 Network Communication

| Protocol | Purpose |
|----------|---------|
| OCPP (Open Charge Point Protocol) | Network management, payment processing, remote monitoring |
| ISO 15118 | Plug & Charge functionality |

## 4. Hardware Components

### 4.1 Connectors

| Standard | Region | Notes |
|----------|--------|-------|
| CCS Combo 1 | North America | Prevailing standard for 800V systems |
| CCS Combo 2 | Europe | Prevailing standard for 800V systems |
| CHAdeMO | Japan/Asia | Alternative standard in some regions |

**Cable Requirements:**
- High-amperage capacity (up to 500A)
- Air-cooled design (passive or fan-assisted)
- Robust construction for frequent use

### 4.2 User Interface

- **Display**: LCD/Touchscreen for user interaction
- **Controls**: Physical start/stop buttons
- **Indicators**: LED status indicators
- **Payment**: Terminal integration capability
- **Accessibility**: Compliance with accessibility standards

## 5. Safety & Protection Systems

### 5.1 Electrical Protections

| Protection Type | Function |
|----------------|----------|
| Over-Voltage Protection (OVP) | Prevents damage from voltage spikes |
| Over-Current Protection (OCP) | Limits current to safe levels |
| Short Circuit Protection (SC) | Immediate shutdown on fault detection |
| Ground Fault Detection | Ensures electrical safety |
| Thermal Management | Prevents overheating of components |

### 5.2 Environmental Specifications

| Parameter | Specification | Notes |
|-----------|---------------|-------|
| Operating Temperature | -20°C to +50°C | Extended range with HVAC |
| Enclosure Rating | IP-rated construction | Weather and dust resistance |
| Cooling System | HVAC clip-on unit | Modular forced-air cooling with refrigeration cycle |
| Cooling Capacity | 15-25 kW thermal | Sized for charger power rating |
| Airflow | 2000-4000 CFM | Variable speed blowers |

## 6. Standards Compliance

### 6.1 Primary Standards

| Standard | Scope |
|----------|-------|
| IEC 61851 | Electrical safety and performance requirements |
| IEC 61851-23/24 | DC charging communication protocols |
| ISO 15118 | Vehicle-to-Grid communication, Plug & Charge |
| OCPP 1.6/2.0.1 | Charge point network protocol |

### 6.2 Safety Certifications

- CE marking (Europe)
- UL certification (North America)
- FCC compliance (electromagnetic compatibility)

## 7. Performance Targets

### 7.1 Charging Speed

- **400V Battery System**: 80% charge in 20-30 minutes
- **800V Battery System**: 80% charge in 15-20 minutes
- **Power Delivery**: Consistent high-power delivery throughout charging session

### 7.2 Reliability

- **Uptime Target**: >98% availability
- **MTBF**: Mean Time Between Failures target
- **Serviceability**: Modular design for easy maintenance
- **HVAC Serviceability**: Clip-on unit allows independent thermal system maintenance

## 8. Related Documentation

- [[__Workspaces/ECG/__init|Main Project Overview]]
- See also: EVerest software framework documentation

## 9. References

- IEC 61851 Standard Series
- ISO 15118 Standard
- Open Charge Point Protocol (OCPP) Specification
- EVerest Project: https://github.com/EVerest/EVerest
