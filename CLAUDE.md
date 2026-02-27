# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a workspace within an Obsidian vault dedicated to **DC Fast Charger (DCFC) Design** research and documentation. The workspace focuses on gathering specifications, technical requirements, and software frameworks related to electric vehicle charging infrastructure.

**Workspace Path:** `/Users/manaspradhan/Library/Mobile Documents/iCloud~md~obsidian/Documents/ClaudeNotes/__Workspaces/DCFC`

**Parent Vault:** ClaudeNotes (Obsidian personal knowledge management vault)

## Content Focus

This workspace contains research and documentation on:

- **Technical Specifications**: Electrical requirements, power conversion (3-phase AC to DC), voltage ranges (150V-1000V DC) for 400V and 800V EV compatibility
- **Standards Compliance**: IEC 61851, ISO 15118, OCPP (Open Charge Point Protocol)
- **Main Controller**: Phytec phyCORE-AM62x (TI AM625, 4x Cortex-A53, 3x native CAN-FD) running Linux + EVerest
- **Hardware Components**: 5x 30 kW PDU-Micro power modules (in-house SiC design), CCS Combo connectors, HVAC clip-on cooling, STM32 safety supervisor
- **Software Framework**: EVerest - open-source modular software framework for EV chargers (Linux Foundation Energy), deployed on Phytec Yocto BSP
- **Communication Protocols**: CAN-FD (power modules), CAN (safety supervisor, HVAC), Ethernet/5G for OCPP network management

## Key Documentation

### `__init.md`
The primary reference document containing:
- Complete DCFC specifications table (input/output, communication, safety)
- EVerest framework overview and architecture
- Microservices architecture using MQTT for inter-process communication
- Industry standards and compliance requirements

## Working with This Workspace

### Creating New Notes
When adding research or documentation:
- Use descriptive filenames related to DCFC components or concepts (e.g., `power-modules.md`, `iso-15118-protocol.md`)
- Link to `__init.md` as the main reference using `[[__init]]`
- Include relevant tags: `#dcfc`, `#everest`, `#standards`, `#hardware`, `#software`
- Cross-reference related concepts using `[[wikilinks]]`

### Note Organization
- Keep technical specifications separate from implementation notes
- Use consistent terminology from industry standards (IEC 61851, ISO 15118)
- When documenting EVerest modules, reference the microservices architecture pattern
- Link hardware specifications to corresponding software components

### Key Terminology
- **DCFC**: DC Fast Charger
- **CCS**: Combined Charging System
- **OCPP**: Open Charge Point Protocol
- **BMS**: Battery Management System
- **EVerest**: Open-source EV charging software framework
- **SiC MOSFETs**: Silicon Carbide transistors for high-efficiency power conversion
- **phyCORE-AM62x**: Phytec SOM used as the main controller (replaced Raspberry Pi CM5)
- **PDU-Micro**: In-house 30 kW SiC power module (5 modules = 150 kW)
- **Safety Supervisor**: Dedicated STM32 controller for hardware interlock and contactor sequencing (SIL 2)

### Important: No CM5 References
The main controller has been migrated from Raspberry Pi CM5 to **Phytec phyCORE-AM62x**. Do not introduce CM5 or Raspberry Pi references in new documentation. The legacy research note `research/02 - CM5 based Main Controller.md` is kept for historical reference only.

## External Resources

- EVerest GitHub: https://github.com/EVerest/EVerest
- Linux Foundation Energy: https://lfenergy.org/projects/everest/
- Phytec phyCORE-AM62x: https://www.phytec.com/product/phycore-am62x/
- Phytec EV Charging Docs: https://docs.phytec.com/projects/yocto-phycore-am62x/en/latest/3rdpartyintegration/ev-charging.html
