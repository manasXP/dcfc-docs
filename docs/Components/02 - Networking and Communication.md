# Networking and Communication

Tags: #dcfc #components #networking #communication

Related: [[docs/Hardware/01 - Hardware Components|01 - Hardware Components]] | [[research/02 - Phytec SBC based Main Controller|02 - Phytec SBC based Main Controller]]

## 1. Overview

The networking and communication subsystem provides local area networking, wide area connectivity for OCPP backend communication, and user authentication. These components enable remote management, payment processing, and diagnostics.

## 2. Components

| Item | Qty | Description | Manufacturer | Model | Est. Price |
|------|-----|-------------|--------------|-------|------------|
| PoE Ethernet Switch | 1 | Industrial 8-port PoE + 2 SFP, managed | Techtonika | TSW200 | $125 |
| 5G Cellular Modem | 1 | Externally mounted, SIM + eSIM, SFP module | Qualcomm | TBD | $200 (est.) |
| RFID/NFC Module | 1 | ISO 14443 / ISO 15693, EMV contactless | — | — | $50 |

## 3. Subtotal

| Category | Total |
|----------|-------|
| **Networking and Communication** | **$375** |

## 4. Notes

- The PoE switch interconnects the Phytec SBC, PLC modems, HMI display, and any IP-based meters/sensors on a local Ethernet backbone
- The 5G modem provides WAN connectivity for OCPP backend, remote diagnostics, SSH/VPN, and log upload
- The RFID/NFC reader handles user authentication (OCPP 2.0.1 Authorization) and contactless payment
- Price for 5G modem is estimated; actual cost TBD based on selected module and carrier requirements
