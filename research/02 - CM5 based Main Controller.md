<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# CM5 

Below is a more detailed **control‑side block diagram** you can give to a hardware designer. It focuses on the CM4, EVSE auxiliary board, PLC, CAN buses, HMI, and backend, consistent with typical EVerest DC charger control architectures.[^1][^2][^3][^4][^5]

## Control‑side block diagram (text)

```text
                 ┌─────────────────────────────────────────────┐
                 │           LOCAL / REMOTE USER               │
                 │                                             │
                 │  - Local HMI Panel (touchscreen)            │
                 │  - Laptop/Tablet (Web UI)                   │
                 │  - OCPP Backend / Cloud                     │
                 └───────────────┬─────────────────────────────┘
                                 │  (Ethernet / 5G / Wi‑Fi)
                                 ▼
         ┌─────────────────────────────────────────────────────────┐
         │      CM4 MAIN CONTROLLER (Raspberry Pi CM4 + IO Board) │
         │────────────────────────────────────────────────────────│
         │  • Linux + EVerest                                     │
         │  • OCPP client, ISO 15118/DIN 70121 stack              │
         │  • Session & power management, logging, diagnostics    │
         │                                                         │
         │  Interfaces:                                            │
         │   - ETH0  → LAN switch / backend / HMI Web UI           │
         │   - 5G Modem (USB/PCIe) → WAN / OCPP                    │
         │   - CAN #1 → DC power bricks (25–30 kW modules)         │
         │   - CAN #2 / UART → EVSE AUX BOARD                      │
         │   - ETH1 / USB → PLC MODEM (for CCS2/NACS)              │
         │   - GPIO / SPI / I2C → misc. IO / monitoring            │
         └───────────────┬────────────────────────────────────────┘
                         │
                         ├───────────────────────────────► HMI
                         │                                (HDMI/DSI/USB)
                         │
                         ├──────── CAN #1 ───────────────► DC POWER BRICKS
                         │
                         ├──────── CAN #2 / UART ────────► EVSE AUX BOARD
                         │
                         └──────── ETH/USB ──────────────► PLC MODEM
```

```text
   ┌──────────────────── EVSE AUXILIARY BOARD ───────────────────┐
   │  (Custom or reference "Yeti"-style EVSE IO board)           │
   │─────────────────────────────────────────────────────────────│
   │  • Interfaces to CM4:                                       │
   │      - CAN or UART link (commands + status)                 │
   │      - Optional digital lines (watchdog, faults, enable)    │
   │                                                             │
   │  • High‑voltage related IO (all isolated):                  │
   │      - Contactor drivers (AC input, DC bus, outlet contactors)
   │      - Pre‑charge relay control                             │
   │      - Feedback: contactor AUX contacts, door/cover switches│
   │      - Emergency stop input                                 │
   │                                                             │
   │  • Safety & measurement interfaces:                         │
   │      - IMD input (OK/fault)                                 │
   │      - DC meter pulses or RS‑485/Ethernet meter link        │
   │                                                             │
   │  • EV connector low‑level interface:                        │
   │      - CP/PP conditioning for CCS2 and NACS                 │
   │      - CP PWM generation and state detection                │
   │      - Proximity/latch switch inputs                        │
   │                                                             │
   │  • Galvanic isolation:                                      │
   │      - Isolated CAN transceivers                            │
   │      - Isolated digital inputs/outputs                      │
   └─────────────────────────────────────────────────────────────┘
```

```text
   ┌──────────────────────── PLC MODEM BLOCK ─────────────────────┐
   │  • PLC chipset (HomePlug Green PHY or similar)               │
   │  • Interface to CM4: Ethernet or USB                         │
   │  • Interface to EV side: CP line coupling for CCS2 & NACS    │
   │  • Used by EVerest ISO 15118/DIN 70121 stack on CM4          │
   └──────────────────────────────────────────────────────────────┘
```

```text
   ┌───────────────────── DC POWER BRICKS (x N) ──────────────────┐
   │  • 25–30 kW modules, paralleled on DC bus                    │
   │  • CAN interface to CM4 (CAN #1)                             │
   │     - Commands: set V, I, enable/disable                     │
   │     - Status: actual V/I/P, temperatures, alarms             │
   └──────────────────────────────────────────────────────────────┘
```


## Signal/communication summary

- **CAN \#1 (high‑power control):**
    - Point‑to‑multipoint between CM4 and all DC power bricks.
    - Used for power setpoint control, module enable/disable and fault monitoring.
- **CAN \#2 or UART (EVSE I/O control):**
    - Between CM4 and EVSE auxiliary board.
    - Used to send high‑level commands (e.g., “close DC contactors”, “enable CP PWM”, “open EV contactors on fault”) and receive status (IMD OK, E‑stop, connector latched, meter values).
- **PLC link:**
    - CM4 ↔ PLC modem via Ethernet/USB.
    - PLC modem ↔ CP lines of CCS2 and NACS connectors.
    - Used for ISO 15118 / DIN 70121 communication to the vehicle.
- **Ethernet / 5G:**
    - CM4 ↔ local network / backend server (OCPP).
    - Optionally also used for remote diagnostics, SSH/VPN, and log upload.
- **HMI:**
    - Either a directly connected panel (HDMI + USB touch) or a thin HMI that just displays a web UI served by EVerest on CM4.


## Components summary


| Component                  | Description                       | Manufacturer | Model Number | Price       | Sources                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------------- | --------------------------------- | ------------ | ------------ | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CM5                        | 4 GB RAM                          | Raspberry Pi |              | $60         | Robu, DigiKey                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| CM5 IO + PoE HAT           | PoE                               | Raspberry Pi |              | $50         | Robu, RoboCraze, DigiKey                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Safety Controller          | STM32Fxxx                         | JouleWorX    |              | $120        | Don't use it if interlocking does the job for electrical faults of all kinds                                                                                                                                                                                                                                                                                                                                                                                     |
| IO Controller              | Industrial Grade, 3 kV Isolation  | Raspberry Pi |              | $80         | https://robocraze.com/products/iriv-ioc-rp2350-ir4-0-industrial-i-o-controller?variant=48197654053088&country=IN&currency=INR&utm_medium=product_sync&utm_source=google&utm_content=sag_organic&utm_campaign=sag_organic&campaignid=22271813913&adgroupid=&keyword=&device=c&gad_source=1&gad_campaignid=22271815110&gbraid=0AAAAADgHQvalKN7uwrc8EfNV0v-XDgOr0&gclid=CjwKCAiAj8LLBhAkEiwAJjbY718O1EI4N2A384ZEuU1OZOcv8_nkMlgl_Q3B4bMPRuAx8wvxLTUdCxoC1cQQAvD_BwE |
| PoE Router                 | TWE 200 8x PoE Port + 2 SFP       | Techtonika   |              | $125        | https://thinkrobotics.com/products/tsw200-industrial-poe-ethernet-switch?variant=49766161219901&country=IN&currency=INR&utm_medium=product_sync&utm_source=google&utm_content=sag_organic&utm_campaign=sag_organic&utm_source=googleads&utm_medium=cpc&gad_source=1&gad_campaignid=23108168791&gbraid=0AAAAACk3EvyjjFdYE0VAwbzmyJisLc66K&gclid=CjwKCAiA7LzLBhAgEiwAjMWzCKs61Q2RgPOty760O-uBdsTsOl1EP_sVmYDTEBPVQykYkLAQr2X7nhoCFgEQAvD_BwE                       |
| 5G Modem/SFP               | Externally Mounted                | Qualcomm     |              | __          | TBD                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Iso CAN + RS485 Controller | 2+1 Isolated CAN x 2 + RS 485 x 1 | Waveshare    |              | $70 x 2     | https://robocraze.com/products/industrial-grade-2-ch-can-to-ethernet-server-2-ch-can-rs485-ethernet-port                                                                                                                                                                                                                                                                                                                                                         |
| PLC Controller x 2         |                                   | PHYTEC       |              | $200 + $200 | PHYTEC                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| PSU 150W AC-DC             | 24V                               | Meanwell     | SDR-150-24   | $30         | SDR-150-24                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| PSU DC-DC                  | 24V, 5V rail                      | Meanwell     | DUPS40       | $30         |                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| LA Battery x 2             | 12v  Maintenance free             | Exide        | 10 Ah        | $20         |                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| RFID/NFC Module            |                                   |              |              | $50         |                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| HD Touch Display           | 14" TFT + Capacitive Touch        | Waveshare    |              | $170        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |


This structure keeps the CM4 focused on **software, protocols, and coordination**, while the EVSE auxiliary board concentrates all **high‑voltage‑related IO and safety signals**, and the CAN‑controlled bricks handle **power conversion**, which matches the modular EVerest charger examples shown in public talks and documentation.[^2][^3][^4][^5][^1]

<div align="center">⁂</div>

[^1]: https://www.youtube.com/watch?v=yVRNRVuncAY

[^2]: https://www.youtube.com/watch?v=7sLQ7IPHuyQ

[^3]: https://archive.fosdem.org/2023/schedule/event/energy_everest/attachments/slides/5341/export/events/attachments/energy_everest/slides/5341/EVerest_FOSDEM2023_Presentation.pdf

[^4]: https://everest.github.io

[^5]: https://static.sched.com/hosted_files/osseu2024/c5/oss_eu24_elc_EVerest.pdf

