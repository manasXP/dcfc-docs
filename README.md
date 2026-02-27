# DC Fast Charger (DCFC) Design Documentation

Design documentation for a 150kW DC Fast Charger supporting both 400V and 800V electric vehicles. Built on the [EVerest](https://github.com/EVerest/EVerest) open-source charging framework.

## Key Specs

| Parameter | Value |
|-----------|-------|
| AC Input | 3-phase 400–480 Vac, 50/60 Hz |
| DC Output | 200–1000 V, up to 150 kW |
| Connector | CCS Combo (liquid-cooled) |
| Efficiency | >96% (SiC MOSFETs) |
| Architecture | Modular power blocks (25/50 kW) |
| Standards | IEC 61851, ISO 15118, OCPP 2.0.1 |

## Documentation

### System
- [System Architecture](docs/System/01%20-%20System%20Architecture.md)
- [Technical Specifications](docs/System/02%20-%20Technical%20Specifications.md)
- [Standards Compliance](docs/System/03%20-%20Standards%20Compliance.md)

### Hardware
- [Hardware Components](docs/Hardware/01%20-%20Hardware%20Components.md)
- [Electric Wiring Diagram](docs/Hardware/02%20-%20Electric%20Wiring%20Diagram.md)
- [Cabinet Layout](docs/Hardware/03%20-%20Cabinet%20Layout.md)
- [Backplane Power Management](docs/Hardware/04%20-%20Backplane%20Power%20Management.md)
- [DC Output Contactor and Pre-Charge Circuit](docs/Hardware/05%20-%20DC%20Output%20Contactor%20and%20Pre-Charge%20Circuit.md)
- [HVAC Clip-On Unit Hardware Design](docs/Hardware/06%20-%20HVAC%20Clip-On%20Unit%20Hardware%20Design.md)
- [CCS Connector and Liquid-Cooled Cable Assembly](docs/Hardware/07%20-%20CCS%20Connector%20and%20Liquid-Cooled%20Cable%20Assembly.md)
- [Power Module Hardware Design](docs/Hardware/08%20-%20Power%20Module%20Hardware%20Design.md)

### HVAC
- [HVAC CANBus Interface Specification](docs/HVAC/04%20-%20HVAC%20CANBus%20Interface%20Specification.md)

### Software
- [Software Framework](docs/Software/01%20-%20Software%20Framework.md)
- [Communication Protocols](docs/Software/02%20-%20Communication%20Protocols.md)
- [Safety Supervisor Controller](docs/Software/03%20-%20Safety%20Supervisor%20Controller.md)
- [Power Module CAN Bus Interface](docs/Software/04%20-%20Power%20Module%20CAN%20Bus%20Interface.md)
- [ISO 15118 Vehicle Communication](docs/Software/05%20-%20ISO%2015118%20Vehicle%20Communication.md)

### EVerest Modules
- [Safety Supervisor Integration](docs/Software/EVerest/01%20-%20EVerest%20Safety%20Supervisor%20Integration.md)
- [Power Module Driver](docs/Software/EVerest/02%20-%20EVerest%20Power%20Module%20Driver.md)
- [OCPP 2.0.1 Backend Integration](docs/Software/EVerest/03%20-%20EVerest%20OCPP201%20Backend%20Integration.md)
- [HVAC Driver](docs/Software/EVerest/04%20-%20EVerest%20HVAC%20Driver.md)
- [Energy Manager](docs/Software/EVerest/05%20-%20EVerest%20Energy%20Manager.md)
- [EvseManager](docs/Software/EVerest/06%20-%20EVerest%20EvseManager.md)
- [Auth Module](docs/Software/EVerest/07%20-%20EVerest%20Auth%20Module.md)

### Research
- [Safety Philosophy](research/01%20-%20Safety%20Philosophy.md)
- [Main Controller (Phytec phyCORE-AM62x)](research/02%20-%20CM5%20based%20Main%20Controller.md)
- [150kW DCFC Comparison](research/03%20-%20150kW%20DCFC%20Comparison.md)
- [Phytec SBC Replacement](research/04%20-%20Phytec%20SBC%20Replacement.md)
- [EVerest Module Architecture](research/05%20-%20EVerest%20Module%20Architecture.md)

## License

This documentation is provided for reference purposes.
