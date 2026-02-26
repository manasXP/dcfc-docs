# HVAC CANBus Interface Specification

Interface specification for the clip-on HVAC unit used in the [[__init|DCFC]] cabinet thermal management system. See [[03 - 150kW DCFC Comparison#Design Decision Air Cooling with Clip-On HVAC Unit]] for the design rationale.

---

## 1. System Context

![[HVAC-CANBus-Connection.excalidraw]]

The HVAC unit is an externally mounted, closed-loop cooling system that maintains the internal cabinet temperature within safe operating limits. It communicates with the [[02 - Phytec SBC based Main Controller|Phytec SBC Main Controller]] over a dedicated CAN bus.

```text
┌──────────────────────────────────────────────────────┐
│              Phytec SBC MAIN CONTROLLER                     │
│                                                      │
│  CAN #1 ──────► DC Power Bricks (25–30 kW modules)  │
│  CAN #2 ──────► EVSE Auxiliary Board                 │
│  CAN #3 ──────► HVAC Unit (this interface)           │
│  ETH/USB ─────► PLC Modem                            │
│  ETH0 ────────► LAN / OCPP Backend                   │
└──────────────────────────────────────────────────────┘
          │
          │ CAN #3 (isolated)
          ▼
┌──────────────────────────────────────────────────────┐
│              CLIP-ON HVAC UNIT                        │
│                                                      │
│  • Compressor + condenser (external loop)            │
│  • Evaporator + fan (internal loop)                  │
│  • Internal MCU (CAN node)                           │
│  • Temperature sensors (cabinet, condenser, evap)    │
│  • Optional heater element (cold climate)            │
└──────────────────────────────────────────────────────┘
```

---

## 2. CAN Bus Physical Layer

| Parameter | Value |
|-----------|-------|
| **Bus Assignment** | CAN #3 (dedicated HVAC bus) |
| **Bitrate** | 250 kbps |
| **Protocol** | CAN 2.0B (extended 29-bit identifiers) |
| **Isolation** | 3 kV galvanic isolation (via isolated CAN transceiver on Phytec SBC IO) |
| **Termination** | 120 Ω at each end (Phytec SBC controller + HVAC unit) |
| **Cable** | Shielded twisted pair, max 5 m |
| **Connector** | 4-pin M12 (CAN_H, CAN_L, GND, Shield) |

> [!note] Why CAN #3?
> CAN #1 handles time-critical power brick control at 500 kbps. CAN #2 handles safety-critical EVSE I/O. A dedicated CAN #3 for HVAC keeps thermal management traffic isolated, preventing bus contention and simplifying fault isolation. The 250 kbps rate is sufficient for the low-frequency HVAC telemetry.

---

## 3. CAN Message Dictionary

### 3.1 Node Addressing

| Node | Address | Role |
|------|---------|------|
| Phytec SBC Main Controller | `0x00` | Bus master |
| HVAC Unit | `0x10` | Slave device |

### 3.2 Message ID Allocation

Base ID format: `0x18XX10YY` where `XX` = message type, `10` = HVAC node, `YY` = sub-index.

| Msg ID (hex) | Direction | Name | Cycle Time | DLC |
|--------------|-----------|------|------------|-----|
| `0x100` | HVAC → Phytec SBC | HVAC_Status_1 | 1000 ms | 8 |
| `0x101` | HVAC → Phytec SBC | HVAC_Status_2 | 1000 ms | 8 |
| `0x102` | HVAC → Phytec SBC | HVAC_Faults | Event-driven | 8 |
| `0x103` | HVAC → Phytec SBC | HVAC_Diagnostics | 5000 ms | 8 |
| `0x200` | Phytec SBC → HVAC | HVAC_Command | 1000 ms | 8 |
| `0x201` | Phytec SBC → HVAC | HVAC_Config | On-demand | 8 |
| `0x7FF` | Both | Heartbeat | 2000 ms | 1 |

---

### 3.3 Message Definitions

#### `0x100` — HVAC_Status_1 (Temperatures)

| Byte | Signal | Unit | Resolution | Range | Description |
|------|--------|------|------------|-------|-------------|
| 0–1 | `cabinet_temp` | °C | 0.1 °C/bit, signed | -40 to +125 °C | Internal cabinet air temperature |
| 2–3 | `condenser_temp` | °C | 0.1 °C/bit, signed | -40 to +125 °C | External condenser coil temperature |
| 4–5 | `evaporator_temp` | °C | 0.1 °C/bit, signed | -40 to +125 °C | Internal evaporator coil temperature |
| 6–7 | `ambient_temp` | °C | 0.1 °C/bit, signed | -40 to +85 °C | External ambient air temperature |

#### `0x101` — HVAC_Status_2 (Operating Data)

| Byte | Signal | Unit | Resolution | Range | Description |
|------|--------|------|------------|-------|-------------|
| 0 | `operating_mode` | enum | — | 0–5 | Current mode (see mode table) |
| 1 | `compressor_speed` | % | 1%/bit | 0–100 | Compressor speed (variable speed drive) |
| 2 | `fan_speed_internal` | % | 1%/bit | 0–100 | Internal circulation fan speed |
| 3 | `fan_speed_external` | % | 1%/bit | 0–100 | External condenser fan speed |
| 4–5 | `power_consumption` | W | 1 W/bit | 0–2000 | Current HVAC power draw |
| 6 | `refrigerant_pressure` | bar | 0.5 bar/bit | 0–40 bar | High-side refrigerant pressure |
| 7 | `hvac_state_flags` | bitfield | — | — | See state flags table |

#### `0x102` — HVAC_Faults (Event-Driven)

Sent immediately on fault detection; repeated every 1000 ms while fault is active.

| Byte | Signal | Description |
|------|--------|-------------|
| 0 | `fault_code` | Primary fault code (see fault table) |
| 1 | `fault_severity` | 0 = Warning, 1 = Minor, 2 = Major, 3 = Critical |
| 2–3 | `fault_value` | Associated value (e.g., temperature at time of fault) |
| 4–7 | `fault_timestamp` | Uptime counter in seconds when fault occurred |

#### `0x103` — HVAC_Diagnostics

| Byte | Signal | Unit | Resolution | Range | Description |
|------|--------|------|------------|-------|-------------|
| 0–3 | `runtime_hours` | hours | 1 hr/bit | 0–4,294,967,295 | Total compressor runtime |
| 4–5 | `compressor_cycles` | count | 1/bit | 0–65,535 | Total compressor start cycles |
| 6–7 | `energy_consumed` | kWh | 0.1 kWh/bit | 0–6,553.5 | Cumulative HVAC energy consumption |

#### `0x200` — HVAC_Command (Phytec SBC → HVAC)

| Byte | Signal | Unit | Resolution | Range | Description |
|------|--------|------|------------|-------|-------------|
| 0 | `mode_request` | enum | — | 0–5 | Requested operating mode |
| 1–2 | `temp_setpoint` | °C | 0.1 °C/bit, signed | +15 to +45 °C | Target cabinet temperature |
| 3 | `fan_override` | % | 1%/bit | 0 = auto, 1–100 = manual | Internal fan speed override |
| 4 | `compressor_enable` | bool | — | 0/1 | Compressor enable (0 = off, 1 = on) |
| 5 | `derating_level` | enum | — | 0–3 | Charger power derating level signalled to HVAC |
| 6–7 | Reserved | — | — | — | — |

#### `0x201` — HVAC_Config (Phytec SBC → HVAC, On-Demand)

| Byte | Signal | Unit | Resolution | Range | Description |
|------|--------|------|------------|-------|-------------|
| 0–1 | `high_temp_alarm` | °C | 0.1 °C/bit | +40 to +80 °C | Cabinet over-temperature alarm threshold |
| 2–3 | `low_temp_alarm` | °C | 0.1 °C/bit | -40 to +10 °C | Cabinet under-temperature alarm threshold |
| 4 | `hysteresis` | °C | 0.5 °C/bit | 1–10 °C | Compressor on/off hysteresis |
| 5 | `defrost_interval` | min | 5 min/bit | 0–255 (0 = auto) | Forced defrost cycle interval |
| 6–7 | Reserved | — | — | — | — |

---

## 4. Enumerations

### Operating Modes (`mode_request` / `operating_mode`)

| Value | Mode | Description |
|-------|------|-------------|
| 0 | `OFF` | HVAC completely off, fans stopped |
| 1 | `STANDBY` | Fans at minimum, compressor off, monitoring only |
| 2 | `COOLING` | Active cooling to maintain setpoint |
| 3 | `HEATING` | Heater element active (cold climate operation) |
| 4 | `DEFROST` | Defrost cycle (reverse refrigerant flow or heater on evaporator) |
| 5 | `VENTILATION` | Fans only, no compressor (mild conditions) |

### State Flags (`hvac_state_flags`, Byte 7 of `0x101`)

| Bit | Flag | Description |
|-----|------|-------------|
| 0 | `compressor_running` | Compressor is currently running |
| 1 | `heater_active` | Heater element is energized |
| 2 | `defrost_active` | Defrost cycle in progress |
| 3 | `filter_warning` | Air filter service interval exceeded |
| 4 | `setpoint_reached` | Cabinet temperature within ±1°C of setpoint |
| 5 | `derate_request` | HVAC requesting charger power derating |
| 6 | `overtemp_warning` | Cabinet temperature above warning threshold |
| 7 | `critical_fault` | Critical fault active — charger should shut down |

### Fault Codes (`fault_code`, Byte 0 of `0x102`)

| Code | Fault | Severity | Action |
|------|-------|----------|--------|
| 0x00 | No fault | — | — |
| 0x01 | Compressor over-current | Major | Disable compressor, retry after cooldown |
| 0x02 | Compressor locked rotor | Major | Disable compressor, require manual reset |
| 0x03 | High-side pressure too high | Major | Disable compressor, check condenser airflow |
| 0x04 | Low-side pressure too low | Minor | Check refrigerant charge |
| 0x05 | Cabinet over-temperature | Critical | Signal Phytec SBC to derate or shut down charger |
| 0x06 | Cabinet under-temperature | Warning | Enable heating mode if available |
| 0x07 | Condenser fan failure | Minor | Derate compressor speed |
| 0x08 | Internal fan failure | Major | Signal Phytec SBC to derate charger |
| 0x09 | Temperature sensor fault | Major | Use backup/default setpoint |
| 0x0A | Communication timeout | Critical | HVAC enters autonomous safe mode |
| 0x0B | Refrigerant leak detected | Critical | Disable compressor, alert operator |
| 0x0C | Heater over-temperature | Major | Disable heater element |
| 0x0D | Power supply fault | Critical | HVAC enters shutdown |

---

## 5. Behaviour & Control Logic

### 5.1 Startup Sequence

```text
1. Phytec SBC powers on → sends Heartbeat on CAN #3
2. HVAC unit powers on → responds with Heartbeat
3. Phytec SBC sends HVAC_Config (0x201) with alarm thresholds
4. Phytec SBC sends HVAC_Command (0x200) with mode = STANDBY, setpoint = 35°C
5. HVAC begins periodic Status_1, Status_2, Diagnostics messages
6. When charger begins a charging session:
   a. Phytec SBC sends mode = COOLING, setpoint = 30°C
   b. HVAC starts compressor and fans
7. When charging session ends:
   a. Phytec SBC sends mode = STANDBY
   b. HVAC ramps down compressor, maintains minimum fan speed
```

### 5.2 Thermal Derating Coordination

```text
           Cabinet Temp
               │
    ≤ 30°C     │    Normal operation (full power)
               │
    30–40°C    │    HVAC in COOLING mode, full compressor
               │
    40–45°C    │    HVAC sets derate_request flag (bit 5)
               │    Phytec SBC reduces charger output by 25%
               │
    45–50°C    │    HVAC sends fault 0x05 (over-temp, Critical)
               │    Phytec SBC reduces charger output by 50%
               │
    > 50°C     │    HVAC sets critical_fault flag (bit 7)
               │    Phytec SBC initiates orderly charger shutdown
```

### 5.3 Communication Loss Handling

| Scenario | Detection | Action |
|----------|-----------|--------|
| Phytec SBC loses HVAC heartbeat | No heartbeat for 6 s (3 missed cycles) | Phytec SBC derates charger to 50%, logs fault, alerts via OCPP |
| HVAC loses Phytec SBC heartbeat | No heartbeat for 6 s | HVAC enters autonomous mode: maintains last setpoint, cooling active |
| Both buses silent > 30 s | HVAC internal watchdog | HVAC maintains cooling at max, sets fault LED on unit |

---

## 6. EVerest Integration

The HVAC interface is implemented as an EVerest module exposing the following:

```text
Module: hvac_controller
├── Provides:
│   ├── cabinet_temperature    (float, °C)
│   ├── ambient_temperature    (float, °C)
│   ├── hvac_state             (enum: OFF, STANDBY, COOLING, HEATING, DEFROST, VENT)
│   ├── hvac_power_draw        (float, W)
│   ├── derate_requested       (bool)
│   └── fault_active           (bool, fault_code)
│
├── Requires:
│   ├── can_interface           (CAN #3 driver)
│   └── energy_manager          (for site-level power budgeting)
│
└── Commands:
    ├── set_mode(mode)
    ├── set_setpoint(temp_c)
    └── get_diagnostics() → {runtime_hrs, cycles, energy_kwh}
```

The module publishes temperature telemetry via MQTT topics:

| Topic | Payload | Rate |
|-------|---------|------|
| `everest/hvac/cabinet_temp` | `{"value": 32.5, "unit": "C"}` | 1 Hz |
| `everest/hvac/ambient_temp` | `{"value": 38.0, "unit": "C"}` | 1 Hz |
| `everest/hvac/status` | `{"mode": "COOLING", "power_w": 350}` | 1 Hz |
| `everest/hvac/fault` | `{"code": "0x05", "severity": "critical"}` | Event |

The energy management module uses `hvac_power_draw` to factor HVAC consumption into total site load calculations, ensuring the combined charger + HVAC draw stays within the grid connection limit.

---

## 7. Hardware Requirements for Phytec SBC Side

| Item | Requirement |
|------|-------------|
| CAN transceiver | Native CAN FD interface on Phytec SBC (phyCORE-AM62x has 3x CAN FD on-chip) |
| CAN channel | CAN #3 — one of the three native CAN FD interfaces on the Phytec SBC |
| Software driver | SocketCAN on Linux (standard kernel driver, TI AM62x CAN FD support) |
| EVerest module | Custom `hvac_controller` module (C++ or Python) |

> [!tip] BOM Impact
> No additional hardware is required. The Phytec phyCORE-AM62x has 3x native CAN FD interfaces. CAN #1 (power modules) and CAN #2 (safety supervisor) use 2 channels; the HVAC interface uses the 3rd native CAN FD channel.

---

## 8. HVAC Unit Requirements

| Parameter | Requirement |
|-----------|-------------|
| Cooling capacity | 1.5–3 kW (selectable by deployment climate) |
| Power supply | 24 Vdc from charger auxiliary PSU, or 230 Vac mains |
| CAN interface | CAN 2.0B, 250 kbps, isolated, M12 connector |
| MCU | Any industrial MCU with CAN peripheral (e.g., STM32F1xx, RP2350) |
| Sensors | 4× NTC/PT1000 (cabinet, condenser, evaporator, ambient) |
| Refrigerant | R-134a or R-290 (propane, for lower GWP) |
| Mounting | Clip-on / bolt-on to cabinet side panel, standard cutout |
| Certifications | CE, UL, IP55 minimum for the HVAC unit itself |

---

#dcfc #hvac #canbus #interface #specification #thermal
