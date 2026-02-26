# EVerest Software Framework

## 1. Overview

EVerest (Electric Vehicle Energy System) is an open-source, modular software framework—often called an "operating system"—for EV chargers. It is managed by the [Linux Foundation Energy](https://lfenergy.org/projects/everest/) and provides a common, customizable base layer for charging station manufacturers.

## 2. Key Characteristics

### 2.1 Open-Source & Modular

- Built on a **microservices architecture**
- Linux processes communicating via MQTT
- Interchangeable modules for different functionalities
- Flexible and extensible design

### 2.2 Comprehensive Stack

EVerest provides a complete software solution including:

| Component | Description |
|-----------|-------------|
| Core Charging Logic | Manages charging sessions and state machines |
| Protocol Support | OCPP, ISO 15118, IEC 61851 |
| Hardware Drivers | Meters, card readers, displays |
| Energy Management | Load balancing, grid integration |
| Simulation Tools | Testing and development environments |

### 2.3 Industry Standardization

- Provides a common base for manufacturers
- Allows focus on value-added features rather than commodity software
- Improves interoperability between different vendors
- Reduces time-to-market for new charger products

## 3. Architecture

### 3.1 Microservices Model

```
┌─────────────────────────────────────────────────────┐
│                 Framework Manager                    │
│         (Module Lifecycle Management)                │
└─────────────────────────────────────────────────────┘
                         │
                    MQTT Broker
                         │
     ┌───────────┬───────┴──────┬───────────┐
     │           │              │           │
┌────┴────┐ ┌────┴────┐   ┌─────┴─────┐ ┌───┴────┐
│ Charging│ │  Energy │   │  Protocol │ │Hardware│
│  Module │ │ Manager │   │  Handler  │ │ Driver │
└─────────┘ └─────────┘   └───────────┘ └────────┘
```

### 3.2 Core Components

| Component | Function |
|-----------|----------|
| **Framework Manager** | Manages module lifecycle, configuration, and startup |
| **MQTT Broker** | Message Queuing Telemetry Transport for inter-process communication |
| **Modules** | Individual software components running as separate processes |
| **Interfaces** | Defined APIs for module communication |

### 3.3 Communication Flow

1. **Loosely Coupled**: Modules operate independently
2. **Message-Based**: All communication through MQTT topics
3. **Asynchronous**: Non-blocking message passing
4. **Scalable**: Easy to add or remove modules

## 4. Protocol Support

### 4.1 Vehicle Communication

| Protocol | Standard | Purpose |
|----------|----------|---------|
| IEC 61851 | Control Pilot | Basic charging control signaling |
| ISO 15118-2 | High-Level Communication | Plug & Charge, smart charging |
| ISO 15118-20 | Next-Gen HLC | Enhanced features, bidirectional charging |
| DIN SPEC 70121 | DC Charging | Legacy DC fast charging |

### 4.2 Network Communication

| Protocol | Version | Purpose |
|----------|---------|---------|
| OCPP | 1.6 | Charge point management |
| OCPP | 2.0.1 | Enhanced features, device management |
| OCPP | 2.1 | ISO 15118 integration, smart charging |

## 5. Module Types

### 5.1 Core Modules

| Module | Description |
|--------|-------------|
| `EvseManager` | Main charging session controller |
| `EnergyManager` | Power distribution and load balancing |
| `Auth` | Authentication and authorization handling |
| `System` | System-level operations and monitoring |

### 5.2 Hardware Interface Modules

| Module | Description |
|--------|-------------|
| `EvseBoard` | Low-level EVSE hardware control |
| `PowerMeter` | Energy metering integration |
| `RFID` | Card reader interfaces |
| `Display` | User interface control |

### 5.3 Protocol Modules

| Module | Description |
|--------|-------------|
| `OCPP` | Backend communication |
| `ISO15118` | High-level vehicle communication |
| `IEC61851` | Basic signaling protocol |

## 6. Development Environment

### 6.1 Prerequisites

- Linux-based operating system
- CMake build system
- C++ compiler (C++17 or later)
- Python 3.x for tooling

### 6.2 Project Structure

```
everest-core/
├── modules/           # EVerest modules
├── interfaces/        # Module interface definitions
├── types/             # Shared type definitions
├── lib/               # Shared libraries
├── config/            # Configuration files
└── docs/              # Documentation
```

### 6.3 Configuration

EVerest uses YAML-based configuration for:
- Module instantiation
- Interface connections
- Runtime parameters
- Hardware mappings

## 7. Benefits for DCFC Development

### 7.1 Reduced Development Time

- Pre-built protocol implementations
- Tested and certified modules
- Community-supported updates

### 7.2 Flexibility

- Modular replacement of components
- Custom module development
- Hardware abstraction layer

### 7.3 Compliance

- Standards-compliant implementations
- Certification-ready code
- Regular updates for new standards

### 7.4 Community Support

- Active development community
- Regular releases and bug fixes
- Industry backing from major manufacturers

## 8. Related Documentation

- [[02 - Technical Specifications]]
- [[__Workspaces/ECG/__init|Main Project Overview]]

## 9. External Resources

- **GitHub Repository**: https://github.com/EVerest/EVerest
- **Linux Foundation Energy**: https://lfenergy.org/projects/everest/
- **Documentation**: https://everest.github.io/
