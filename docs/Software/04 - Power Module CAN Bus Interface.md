# Power Module CAN Bus Interface

Tags: #dcfc #can #power-modules #software #hardware

Related: [[01 - EVerest Safety Supervisor Integration]] | [[01 - Software Framework]] | [[docs/System/01 - System Architecture|01 - System Architecture]] | [[docs/Hardware/01 - Hardware Components|01 - Hardware Components]]

## 1. Overview

The DC power modules (25 kW bricks) communicate with the Phytec SBC main controller over a dedicated CAN bus (CAN #1). This interface handles voltage/current setpoint control, module enable/disable, status telemetry, and fault reporting. Each module contains its own DSP/MCU that runs closed-loop power conversion; the Phytec SBC acts as the bus master, issuing high-level setpoints and monitoring aggregate output.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Phytec SBC (Linux + EVerest)                           │
│                                                                         │
│  ┌───────────────┐     ┌──────────────────────────────────────────────┐ │
│  │  EvseManager  │────▶│  PowerModuleDriver (EVerest C++ module)      │ │
│  │               │     │  provides: power_supply_DC interface         │ │
│  │  setMode()    │     │  provides: powermeter interface (optional)   │ │
│  │  setExport    │     │                                              │ │
│  │  VoltageCurr  │     │  Translates EVerest commands → CAN frames    │ │
│  │               │     │  Aggregates module status → EVerest vars     │ │
│  └───────────────┘     └──────────────────┬───────────────────────────┘ │
│                                           │                             │
└───────────────────────────────────────────┼─────────────────────────────┘
                                            │ CAN #1 (500 kbps)
                                            │
            ┌───────────────────────────────┼──────────────────────────┐
            │                 |                 │                      │
     ┌──────┴──────┐   ┌──────┴──────┐   ┌──────┴──────┐        ┌──────┴──────┐
     │  Module 1   │   │  Module 2   │   │  Module 3   │  ...   │  Module N   │
     │   25 kW     │   │   25 kW     │   │   25 kW     │        │   25 kW     │
     │  Node 0x01  │   │  Node 0x02  │   │  Node 0x03  │        │  Node 0x0N  │
     └─────────────┘   └─────────────┘   └─────────────┘        └─────────────┘
            │                 │                 │                      │
            └─────────────────┴─────────────────┴──────────────────────┘
                                    DC OUTPUT BUS
                                   (paralleled outputs)
```

## 2. CAN Bus Physical Layer

| Parameter | Value |
|-----------|-------|
| **Bus Assignment** | CAN #1 (dedicated power module bus) |
| **Bitrate** | 500 kbps |
| **Protocol** | CAN 2.0A (standard 11-bit identifiers) |
| **Isolation** | 3 kV galvanic isolation (isolated CAN transceiver on Phytec SBC side) |
| **Termination** | 120 Ω at each end (Phytec SBC controller + last module on bus) |
| **Cable** | Shielded twisted pair, daisy-chain topology |
| **Max Bus Length** | 20 m (at 500 kbps, all modules within single cabinet) |
| **Max Nodes** | 12 modules + 1 master (Phytec SBC) |

> [!note] Why CAN #1 is separate
> Power module control requires faster cycle times (50–100 ms) than HVAC telemetry (CAN #3, 250 kbps) and must not share bus bandwidth with safety-critical EVSE I/O (CAN #2). A dedicated 500 kbps bus ensures deterministic setpoint updates across all modules.

### 2.1 Bus Topology

```
Phytec SBC (Master)           Module 1           Module 2           Module N
  ┌───┐                 ┌───┐              ┌───┐              ┌───┐
  │120Ω│                │   │              │   │              │120Ω│
  └─┬─┘                 │   │              │   │              └─┬─┘
────┴───────────────────┴───┴──────────────┴───┴──────────────┴──── CAN_H
────┬───────────────────┬───┬──────────────┬───┬──────────────┬──── CAN_L
    │                   │   │              │   │              │
   GND                 GND GND            GND GND            GND

Daisy-chain wiring: Phytec SBC → Module 1 → Module 2 → ... → Module N
Termination only at Phytec SBC and last module on the chain.
```

## 3. Node Addressing

Each power module has a hardware-configurable node address set via DIP switches or resistor strapping on its controller board.

| Node ID | Assignment | Notes |
|---------|------------|-------|
| `0x00` | Phytec SBC Main Controller (bus master) | Source address for all commands |
| `0x01` – `0x0C` | Power Modules 1–12 | Individually addressable |
| `0x3F` | Broadcast | Commands to all modules simultaneously |

## 4. CAN Message Dictionary

### 4.1 Message ID Scheme

Standard 11-bit CAN IDs are structured as:

```
Bits 10-8: Message group (0 = command, 1 = status, 2 = telemetry, 3 = fault)
Bits  7-4: Node ID (0x0 = master, 0x1–0xC = modules, 0xF = broadcast)
Bits  3-0: Message sub-index within group
```

| ID Range | Direction | Group | Description |
|----------|-----------|-------|-------------|
| `0x010` – `0x01F` | Phytec SBC → Module | Command | Setpoints, enable/disable |
| `0x0F0` – `0x0FF` | Phytec SBC → All | Command (broadcast) | Global commands |
| `0x110` – `0x1CF` | Module → Phytec SBC | Status | Per-module state and measurements |
| `0x210` – `0x2CF` | Module → Phytec SBC | Telemetry | Temperatures, efficiency, runtime |
| `0x310` – `0x3CF` | Module → Phytec SBC | Fault | Per-module fault codes and details |

### 4.2 Command Messages (Phytec SBC → Module)

#### `0x010` + Node Offset: Set Voltage/Current Setpoint

Sent to individual modules or broadcast (`0x0F0`).

| Byte | Field | Type | Unit | Range | Description |
|------|-------|------|------|-------|-------------|
| 0–1 | Voltage setpoint | uint16 LE | 0.1 V | 0–10000 (0–1000.0 V) | Target output voltage |
| 2–3 | Current limit | uint16 LE | 0.1 A | 0–5000 (0–500.0 A) | Per-module current limit |
| 4 | Mode | uint8 | enum | 0–4 | See mode table below |
| 5 | Ramp rate | uint8 | V/ms | 0–255 | Voltage slew rate (0 = default) |
| 6–7 | Reserved | — | — | — | Set to `0x00` |

**Mode values:**

| Value | Mode | Description |
|-------|------|-------------|
| `0x00` | Off | Module disabled, output off |
| `0x01` | Standby | Module powered, output disabled, ready to enable |
| `0x02` | Voltage Control (CV) | Regulate to voltage setpoint, current limited |
| `0x03` | Current Control (CC) | Regulate to current limit, voltage clamped |
| `0x04` | Precharge | Low-current voltage ramp for DC bus precharge |

**Cycle time:** 100 ms (broadcast), or on-change for individual module adjustments.

#### `0x011` + Node Offset: Module Control

| Byte | Field | Type | Description |
|------|-------|------|-------------|
| 0 | Enable | uint8 | 0 = disable, 1 = enable output |
| 1 | Fan override | uint8 | 0 = auto, 1–100 = manual fan speed (%) |
| 2 | Clear faults | uint8 | 1 = clear latched faults and attempt restart |
| 3 | Priority | uint8 | 0 = normal, 1 = reduced (for N+1 standby) |
| 4–7 | Reserved | — | Set to `0x00` |

#### `0x0F2`: Global Emergency Disable (Broadcast)

| Byte | Field | Type | Description |
|------|-------|------|-------------|
| 0 | Emergency disable | uint8 | 1 = immediate output disable (all modules) |
| 1–7 | Reserved | — | Set to `0x00` |

This message is sent by the Phytec SBC when EvseManager calls `powersupply_DC.setMode(Off)` or upon receiving a fault from the safety supervisor. All modules must disable output within 10 ms of receiving this frame.

### 4.3 Status Messages (Module → Phytec SBC)

#### `0x110` + Node Offset: Module Status

Sent periodically by each module.

| Byte | Field | Type | Unit | Description |
|------|-------|------|------|-------------|
| 0–1 | Output voltage | uint16 LE | 0.1 V | Measured output voltage |
| 2–3 | Output current | uint16 LE | 0.1 A | Measured output current |
| 4 | Module state | uint8 | enum | See state table below |
| 5 | Fault flags (low) | uint8 | bitmask | Active fault indicators |
| 6 | Power limit (%) | uint8 | % | Current derating level (100 = full) |
| 7 | Fan speed (%) | uint8 | % | Measured fan duty cycle |

**Module state values:**

| Value | State | Description |
|-------|-------|-------------|
| `0x00` | Init | Module booting, self-test in progress |
| `0x01` | Ready | Self-test passed, output off, awaiting enable |
| `0x02` | Standby | Enabled, DC link charged, output disabled |
| `0x03` | CV Mode | Voltage regulation active |
| `0x04` | CC Mode | Current regulation active |
| `0x05` | Derating | Output active but power-limited (thermal) |
| `0x06` | Fault | Output disabled, fault latched |
| `0x07` | Shutdown | Controlled shutdown in progress |

**Cycle time:** 100 ms per module. With 6 modules, the Phytec SBC receives ~60 status messages/second on CAN #1.

#### `0x111` + Node Offset: Power Measurement

Higher-resolution measurement data for energy metering.

| Byte | Field | Type | Unit | Description |
|------|-------|------|------|-------------|
| 0–1 | Output voltage | uint16 LE | 0.01 V | High-resolution voltage |
| 2–3 | Output current | uint16 LE | 0.01 A | High-resolution current |
| 4–5 | Output power | uint16 LE | 0.1 kW | Calculated output power |
| 6–7 | Energy delivered | uint16 LE | 0.01 kWh | Session energy counter |

**Cycle time:** 1000 ms. Used by the EVerest `powermeter` interface for billing and OCPP `MeterValues`.

### 4.4 Telemetry Messages (Module → Phytec SBC)

#### `0x210` + Node Offset: Thermal Telemetry

| Byte | Field | Type | Unit | Description |
|------|-------|------|------|-------------|
| 0 | MOSFET temperature | int8 | °C | SiC MOSFET heatsink temp |
| 1 | Transformer temperature | int8 | °C | HF transformer winding temp |
| 2 | Inductor temperature | int8 | °C | PFC inductor core temp |
| 3 | DC link cap temperature | int8 | °C | DC link capacitor case temp |
| 4 | Inlet air temperature | int8 | °C | Module air inlet temp |
| 5 | PCB temperature | int8 | °C | Controller board temp |
| 6 | Efficiency (%) | uint8 | 0.5% steps | Current operating efficiency (0–127 → 0–63.5%) not used; (128–255 → 64–127.5%) typical range 190–196 → 95–98% |
| 7 | Runtime hours (high byte) | uint8 | 256 h | Combined with low byte in diagnostics |

**Cycle time:** 2000 ms.

#### `0x211` + Node Offset: Electrical Telemetry

| Byte | Field | Type | Unit | Description |
|------|-------|------|------|-------------|
| 0–1 | AC input voltage (L-L) | uint16 LE | 0.1 V | Input line-to-line voltage |
| 2–3 | AC input current | uint16 LE | 0.1 A | RMS input current |
| 4–5 | DC link voltage | uint16 LE | 0.1 V | Internal DC link bus voltage |
| 6 | Power factor | uint8 | 0.01 | Power factor × 100 (e.g., 99 = 0.99) |
| 7 | THD (%) | uint8 | 0.1% | Total harmonic distortion × 10 |

**Cycle time:** 2000 ms.

### 4.5 Fault Messages (Module → Phytec SBC)

#### `0x310` + Node Offset: Fault Report

Sent on fault event and repeated every 1000 ms while fault is active.

| Byte | Field | Type | Description |
|------|-------|------|-------------|
| 0 | Fault code | uint8 | See fault table below |
| 1 | Fault severity | uint8 | 0 = warning, 1 = major, 2 = critical |
| 2–3 | Fault value | uint16 LE | Measured value that triggered the fault (unit depends on fault type) |
| 4–5 | Timestamp | uint16 LE | Module uptime in seconds when fault occurred |
| 6 | Fault flags (expanded) | uint8 | Bitmask of all active faults |
| 7 | Restart attempts | uint8 | Auto-restart counter since last clear |

**Module fault codes:**

| Code | Fault | Value Unit | Severity | Auto-Restart |
|------|-------|------------|----------|--------------|
| `0x01` | Output overvoltage | 0.1 V | Critical | No |
| `0x02` | Output overcurrent | 0.1 A | Critical | No |
| `0x03` | DC link overvoltage | 0.1 V | Critical | No |
| `0x04` | DC link undervoltage | 0.1 V | Major | Yes (3 attempts) |
| `0x05` | AC input undervoltage | 0.1 V | Major | Yes (3 attempts) |
| `0x06` | AC input overvoltage | 0.1 V | Major | Yes (3 attempts) |
| `0x07` | MOSFET over-temperature | °C | Major | Yes (on cooldown) |
| `0x08` | Transformer over-temperature | °C | Major | Yes (on cooldown) |
| `0x09` | Capacitor over-temperature | °C | Major | Yes (on cooldown) |
| `0x0A` | Fan failure | RPM | Warning | — |
| `0x0B` | Short circuit | — | Critical | No |
| `0x0C` | PFC fault | — | Critical | No |
| `0x0D` | Communication timeout | ms | Warning | — |
| `0x0E` | Internal self-test fail | test ID | Critical | No |
| `0x0F` | Current imbalance (phases) | 0.1 A | Warning | — |

### 4.6 Heartbeat

Each module sends a heartbeat frame at 1000 ms intervals. The Phytec SBC monitors all modules and flags any module that misses 3 consecutive heartbeats as offline.

#### `0x700` + Node ID: Module Heartbeat

| Byte | Field | Type | Description |
|------|-------|------|-------------|
| 0 | NMT state | uint8 | 0x00 = boot, 0x04 = stopped, 0x05 = operational, 0x7F = pre-operational |

This follows the CANopen NMT heartbeat convention for compatibility with standard CAN tools and analyzers.

#### `0x700`: Master Heartbeat (Phytec SBC)

| Byte | Field | Type | Description |
|------|-------|------|-------------|
| 0 | Rolling counter | uint8 | Incrementing counter (0–255) |

Modules monitor the master heartbeat. If lost for >2 s, each module autonomously ramps current to zero and enters standby. This is independent of the safety supervisor's own watchdog on CAN #2.

## 5. Control Modes and Sequencing

### 5.1 Normal Charging Sequence

```
Time ─────────────────────────────────────────────────────────────────▶

1. PREPARE       Phytec SBC sends: Mode = Standby, Enable = 1 (per module)
                 Modules: DC link charges, output remains off
                 Status: each module reports state = 0x02 (Standby)

2. PRECHARGE     Phytec SBC sends: Mode = Precharge, V = target, I = 2A
                 (broadcast 0x0F0)
                 Modules: one designated module ramps voltage to target
                 Phytec SBC monitors: V_out approaches V_target via status 0x110

3. FULL CHARGE   Phytec SBC sends: Mode = CV, V = target, I = per-module limit
                 (broadcast 0x0F0)
                 All modules: parallel output, current sharing
                 Each module regulates to the same voltage setpoint
                 Current limit split: total_I / active_modules

4. CC PHASE      When battery nears full voltage:
                 Phytec SBC sends: Mode = CC (EV requests constant current)
                 Modules: all in CC mode, voltage floats to battery

5. TAPER         Phytec SBC reduces I setpoint progressively per EV request
                 Modules: follow ramp-down, some may go to Standby
                 Individual modules can be shed for efficiency at low power

6. SHUTDOWN      Phytec SBC sends: Mode = Off (broadcast 0x0F0)
                 All modules: ramp current to zero, disable output
                 Status: each module reports state = 0x01 (Ready)
```

### 5.2 Current Sharing

All modules connected to the shared DC output bus operate in droop-mode current sharing, coordinated by the Phytec SBC:

```
Total requested current: 300 A
Active modules: 6 × 25 kW
Per-module limit: 300 / 6 = 50 A each

Phytec SBC sends to each module:
  Voltage setpoint: 800.0 V (same for all)
  Current limit: 50.0 A (equal split)

If Module 3 is derated to 70% (thermal):
  Module 3 limit: 35 A
  Remaining 5 modules: (300 - 35) / 5 = 53 A each
  Phytec SBC recalculates and re-sends individual setpoints
```

The Phytec SBC recalculates current distribution whenever:
- A module reports derating (power limit < 100%)
- A module goes offline or enters fault
- The EV changes its current demand
- EnergyManager changes the site power limit

### 5.3 N+1 Redundancy

One module is designated as hot standby (Priority = 1). It stays in Standby state with DC link charged but output disabled. If an active module faults:

```
1. Module 4 reports Fault (0x310, code 0x07, MOSFET over-temperature)
2. Phytec SBC receives fault, removes Module 4 from active pool
3. Phytec SBC recalculates: 5 remaining modules carry the load
4. If total demand > 5-module capacity:
   Phytec SBC enables standby module (Module 7, Priority → 0, Enable → 1)
5. Phytec SBC sends updated current limits to all active modules
6. EvseManager logs event, OCPP sends StatusNotification

Total interruption: <200 ms (standby module DC link already charged)
```

### 5.4 Graceful Degradation

| Active Modules | Max Output | Action |
|----------------|------------|--------|
| 6 (all) | 150 kW | Normal operation |
| 5 | 125 kW | Standby promoted, minor derating if needed |
| 4 | 100 kW | EvseManager reports reduced capacity to EV |
| 3 | 75 kW | ISO 15118 renegotiation of charge parameters |
| 2 | 50 kW | Continued charging at reduced rate |
| 1 | 25 kW | Minimum viable charging |
| 0 | 0 kW | Session terminated, fault reported |

## 6. EVerest Integration

### 6.1 power_supply_DC Interface

The `PowerModuleDriver` EVerest module implements the `power_supply_DC` interface, which EvseManager uses to control the DC output.

**Commands (EvseManager → PowerModuleDriver):**

| Command | Arguments | Description |
|---------|-----------|-------------|
| `setMode` | `mode: string, reason: string` | Set operating mode: `Off`, `Export`, `Import`, `Fault`, `Precharge` |
| `setExportVoltageCurrent` | `voltage: number, current: number` | Target voltage (V) and total current limit (A) |
| `setImportVoltageCurrent` | `voltage: number, current: number` | For V2G bidirectional (future) |
| `getCapabilities` | — | Returns min/max V/I/P per connector |

**Variables (PowerModuleDriver → EvseManager):**

| Variable | Type | Description |
|----------|------|-------------|
| `voltage_current` | `{voltage_V, current_A}` | Measured aggregate output |
| `mode` | `string` | Current operating mode |
| `capabilities` | `DCSupplyCapabilities` | Hardware limits |

### 6.2 Module Manifest

```yaml
# modules/PowerModuleDriver/manifest.yaml
description: >
  CAN-based driver for paralleled 25 kW DC power modules.
  Provides power_supply_DC and powermeter interfaces.
config:
  can_device:
    description: Linux SocketCAN interface name
    type: string
    default: can1
  num_modules:
    description: Number of power modules on the bus
    type: integer
    default: 6
  module_power_kw:
    description: Rated power per module in kW
    type: number
    default: 25.0
  max_voltage_V:
    description: Maximum output voltage
    type: number
    default: 1000.0
  max_current_A:
    description: Maximum total output current
    type: number
    default: 500.0
  heartbeat_timeout_ms:
    description: Module heartbeat timeout
    type: integer
    default: 3000
  redundant_modules:
    description: Number of N+1 standby modules
    type: integer
    default: 1
provides:
  main:
    interface: power_supply_DC
    description: DC power output control
  powermeter:
    interface: powermeter
    description: Aggregated energy metering
```

### 6.3 YAML Configuration Wiring

```yaml
active_modules:

  powersupply_dc:
    module: PowerModuleDriver
    config_module:
      can_device: can1
      num_modules: 6
      module_power_kw: 25.0
      max_voltage_V: 1000.0
      max_current_A: 375.0      # 6 × 62.5 A per module at 400V
      heartbeat_timeout_ms: 3000
      redundant_modules: 1

  evse_manager:
    module: EvseManager
    config_module:
      charge_mode: DC
    connections:
      powersupply_DC:
        - module_id: powersupply_dc
          implementation_id: main
      powermeter_car_side:          # optional: use module-reported energy
        - module_id: powersupply_dc
          implementation_id: powermeter
```

### 6.4 Driver Logic Overview

```cpp
// PowerModuleDriver.cpp — simplified control loop

void PowerModuleDriver::handle_setExportVoltageCurrent(
        double voltage_V, double current_A) {
    // Calculate per-module current based on active/healthy modules
    auto active = get_active_modules();  // excludes faulted and standby
    double per_module_I = current_A / active.size();

    // Apply derating adjustments
    for (auto& mod : active) {
        double adjusted_I = std::min(per_module_I, mod.available_current());
        double surplus = per_module_I - adjusted_I;
        // Redistribute surplus to other modules
        redistribute_current(active, surplus, mod.id);
    }

    // Send CAN setpoints
    for (auto& mod : active) {
        uint8_t frame[8] = {};
        pack_u16_le(frame, 0, static_cast<uint16_t>(voltage_V * 10));
        pack_u16_le(frame, 2, static_cast<uint16_t>(mod.current_limit * 10));
        frame[4] = static_cast<uint8_t>(current_mode_);
        can_.tx(0x010 + mod.node_id, frame, 8);
    }
}

void PowerModuleDriver::on_module_status(uint8_t node_id, const uint8_t* data) {
    auto& mod = modules_[node_id];
    mod.voltage_V    = unpack_u16_le(data, 0) / 10.0;
    mod.current_A    = unpack_u16_le(data, 2) / 10.0;
    mod.state        = data[4];
    mod.fault_flags  = data[5];
    mod.power_limit  = data[6];
    mod.fan_speed    = data[7];

    // Aggregate for EVerest
    double total_V = aggregate_voltage();   // average of all active modules
    double total_I = aggregate_current();   // sum of all active modules
    publish_voltage_current({total_V, total_I});

    // Handle degradation
    if (mod.state == 0x06) {  // Fault
        handle_module_fault(node_id);
    } else if (mod.power_limit < 100) {
        recalculate_current_distribution();
    }
}
```

## 7. Timing and Bandwidth Analysis

### 7.1 Message Schedule

| Message | Direction | Count | Period | Frames/sec |
|---------|-----------|-------|--------|------------|
| Setpoint (broadcast) | Phytec SBC → All | 1 | 100 ms | 10 |
| Module Control | Phytec SBC → Individual | ~2 | On-change | ~2 |
| Module Status | Module → Phytec SBC | 6 | 100 ms | 60 |
| Power Measurement | Module → Phytec SBC | 6 | 1000 ms | 6 |
| Thermal Telemetry | Module → Phytec SBC | 6 | 2000 ms | 3 |
| Electrical Telemetry | Module → Phytec SBC | 6 | 2000 ms | 3 |
| Heartbeat | Both | 7 | 1000 ms | 7 |
| Fault (worst case) | Module → Phytec SBC | 6 | 1000 ms | 6 |
| **Total** | | | | **~97** |

At 500 kbps with an average frame of 130 bits (including overhead): 97 frames/s × 130 bits = 12,610 bits/s → **2.5% bus utilization**. Well within safe limits (recommended <50% for real-time systems).

### 7.2 Response Time Budget

| Event | Max Latency | Constraint |
|-------|-------------|------------|
| Setpoint update → module response | 100 ms | Status update confirms new setpoint |
| Emergency disable → output off | 10 ms | Module must disable within 1 CAN frame + processing |
| Module fault → Phytec SBC notification | 100 ms | Next status cycle |
| Module fault → current redistribution | 200 ms | Phytec SBC recalculate + broadcast new setpoints |
| Heartbeat loss → standby promotion | 3 s | 3 missed heartbeats + enable sequence |

## 8. Module Boot and Discovery

### 8.1 Startup Sequence

```
1. Phytec SBC powers on, initializes CAN #1 interface
2. Phytec SBC sends master heartbeat (0x700) at 1000 ms intervals
3. Each power module boots independently:
   a. Self-test (ADC, PWM, gate drivers, temperature sensors)
   b. Module sends heartbeat (0x701–0x70C) with NMT state = 0x00 (boot)
   c. Self-test pass → NMT state = 0x7F (pre-operational)
4. Phytec SBC detects modules via heartbeat, builds active module list
5. Phytec SBC sends Module Control with Enable = 0, Priority per config
6. Modules transition to NMT state = 0x05 (operational), report Ready (0x01)
7. Phytec SBC reports capabilities to EvseManager via power_supply_DC interface
```

### 8.2 Hot-Swap Detection

If a module heartbeat disappears and later reappears:

```
1. Module heartbeat lost → Phytec SBC marks module as offline
2. Current redistributed across remaining modules
3. Module heartbeat returns (NMT = 0x00, then 0x7F)
4. Phytec SBC sends Enable = 0, queries module health via status
5. If healthy → add to standby pool or active pool
6. Recalculate current distribution
```

This supports hot-swap maintenance without stopping a charging session.

## 9. Interaction with Safety Supervisor

The power module CAN bus (CAN #1) is managed by the Phytec SBC, but the safety supervisor on CAN #2 has indirect authority:

```
┌──────────────────────────────────────────────────────────────┐
│  Safety Supervisor (STM32, CAN #2)                           │
│                                                              │
│  Cannot talk to power modules directly (separate CAN bus).   │
│  Controls power modules indirectly via:                      │
│                                                              │
│  1. ENABLE signal (DO5): hardware line to all modules        │
│     If safety supervisor de-asserts ENABLE, all modules      │
│     must disable output regardless of CAN commands.          │
│                                                              │
│  2. Phytec SBC watchdog: if Phytec SBC dies, safety supervisor opens       │
│     contactors. Modules also detect lost master heartbeat    │
│     and ramp to zero independently.                          │
│                                                              │
│  3. Hardware interlock chain (Layer 1): if tripped, AC       │
│     power to modules is cut by the AC contactor.             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

The ENABLE hardware signal provides a safety-rated shutdown path that bypasses both CAN buses entirely. Each power module has a hardware input pin: if ENABLE is not asserted, the module must disable gate drive signals within 1 ms regardless of the CAN command state.

## 10. Related Documentation

- [[01 - EVerest Safety Supervisor Integration]] — How EvseManager uses `power_supply_DC` interface
- [[03 - Safety Supervisor Controller]] — ENABLE signal and safety shutdown paths
- [[01 - Software Framework]] — EVerest module architecture and MQTT IPC
- [[02 - Communication Protocols]] — CAN bus physical layer and wiring
- [[docs/System/01 - System Architecture|01 - System Architecture]] — Modular power architecture and 25 kW module internals
- [[docs/Hardware/01 - Hardware Components|01 - Hardware Components]] — Power module electrical specifications
- [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]] — CAN #3 HVAC interface (design pattern reference)
