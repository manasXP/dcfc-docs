# EVerest Safety Supervisor Integration

Tags: #dcfc #everest #safety #software #integration #can

Related: [[03 - Safety Supervisor Controller]] | [[01 - Software Framework]] | [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]]

## 1. Overview

This document describes how the EVerest software framework running on the CM5 integrates with the STM32-based safety supervisor controller. The two systems have distinct roles: EVerest owns session logic, protocols, and energy management; the safety supervisor owns contactor control, CP/PP hardware, and fault enforcement. They communicate over CAN bus, with the safety supervisor implementing a custom EVerest Board Support Package (BSP) driver module on the CM5 side.

```
┌───────────────────────────────────────────────────────────────────────┐
│                         CM5 (Linux + EVerest)                         │
│                                                                       │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │  EvseManager │  │    Auth     │  │     OCPP     │  │  Energy   │  │
│  │  (session    │  │  (tokens,   │  │  (backend    │  │  Manager  │  │
│  │   state      │  │   RFID)     │  │   comms)     │  │  (limits) │  │
│  │   machine)   │  │             │  │              │  │           │  │
│  └──────┬───────┘  └─────────────┘  └──────────────┘  └───────────┘  │
│         │ requires: bsp, imd                                          │
│         ▼                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │              SafetySupervisorBSP (custom C++ module)             │ │
│  │                                                                  │ │
│  │  provides:                                                       │ │
│  │    board_support  → evse_board_support interface                  │ │
│  │    imd            → isolation_monitor interface                   │ │
│  │    rcd            → ac_rcd interface                              │ │
│  │    connector_lock → connector_lock interface                      │ │
│  │                                                                  │ │
│  │  Translates EVerest commands → CAN frames                        │ │
│  │  Translates CAN frames → EVerest events/variables                │ │
│  └──────────────────────────────┬───────────────────────────────────┘ │
│                                 │                                     │
└─────────────────────────────────┼─────────────────────────────────────┘
                                  │ CAN #2 (500 kbps)
                                  │
┌─────────────────────────────────┼─────────────────────────────────────┐
│                    SAFETY SUPERVISOR (STM32)                           │
│                                                                       │
│  Receives: enable, PWM duty, allow_power_on, lock/unlock              │
│  Sends:    CP state, contactor feedback, IMD values, faults           │
│                                                                       │
│  Independently enforces: OVP, OCP, watchdog, interlock chain          │
└───────────────────────────────────────────────────────────────────────┘
```

## 2. Integration Architecture

### 2.1 Why a Custom BSP Module

EVerest's `EvseManager` does not control hardware directly. It depends on a module that implements the `evse_board_support` interface. In our design, the safety supervisor handles all low-level hardware (CP generation, contactor drivers, IMD readout), so the BSP module on the CM5 is a **CAN translator** — not a hardware driver in the traditional sense.

Alternatives considered:

| Approach | Description | Why Not |
|----------|-------------|---------|
| `evse_board_support_API` (MQTT bridge) | Safety supervisor connects to MQTT broker directly | Adds MQTT client to safety-critical MCU; increases attack surface and complexity |
| YetiDriver-style serial (COBS + Protobuf) | Serial link with nanopb framing | CAN preferred for noise immunity in HV cabinet; serial viable as fallback |
| **Custom CAN BSP module** | C++ module on CM5 using SocketCAN | **Selected** — cleanest separation, leverages existing CAN #2 bus |

### 2.2 Module Responsibilities Split

| Function | EVerest (CM5) | Safety Supervisor (STM32) |
|----------|---------------|---------------------------|
| CP PWM generation | Sends duty cycle value | Generates 1 kHz ±12V PWM on hardware |
| CP state detection | Receives state via CAN | Reads CP voltage, classifies A-F |
| Contactor control | Sends `allow_power_on` | Validates, sequences contactors |
| Contactor feedback | Receives PowerOn/PowerOff events | Reads AUX contacts, detects welds |
| PP cable rating | Receives ampacity value | Reads PP resistor, classifies |
| IMD monitoring | Sends start/stop/self_test | Reads IMD relay + analog value |
| RCD monitoring | Receives current + trip events | Reads RCD relay status |
| Connector lock | Sends lock/unlock | Drives lock motor, reads feedback |
| OVP/OCP | Monitors via reported values | Independent ADC check, trips directly |
| Watchdog | Sends heartbeat (CAN `0x100`) | Trips on missed heartbeat |
| Session logic | Full ownership (EvseManager) | None — follows commands |
| ISO 15118 / OCPP | Full ownership | None |

## 3. CAN Protocol Mapping to EVerest Interfaces

### 3.1 evse_board_support Commands (CM5 → STM32)

The BSP module translates each EVerest command into a CAN frame on CAN #2.

| EVerest Command | CAN ID | Payload | Notes |
|-----------------|--------|---------|-------|
| `enable(bool)` | `0x110` | Byte 0: 0=disable, 1=enable | Disabling blocks new sessions |
| `pwm_on(duty%)` | `0x111` | Bytes 0-1: duty in 0.01% units (uint16, LE) | 0 = State F, 10001 = X1 |
| `cp_state_X1()` | `0x111` | Bytes 0-1: `0x2711` (10001) | Sentinel value for standby |
| `cp_state_F()` | `0x111` | Bytes 0-1: `0x0000` | Sentinel value for −12V |
| `allow_power_on(PowerOnOff)` | `0x101` | Byte 0: 0=off, 1=on; Byte 1: reason enum | Maps to Enable Request |
| `ac_set_overcurrent_limit_A(A)` | `0x112` | Bytes 0-1: current in 0.1A units (uint16, LE) | Hardware trip threshold |
| Lock: `lock()` | `0x113` | Byte 0: 1 | Connector latch motor |
| Lock: `unlock()` | `0x113` | Byte 0: 0 | Connector latch motor |

### 3.2 evse_board_support Variables (STM32 → CM5)

The BSP module subscribes to CAN frames from the safety supervisor and publishes them as EVerest variables.

| CAN ID | EVerest Variable | Payload | Publish Trigger |
|--------|-----------------|---------|-----------------|
| `0x210` | `event` (BspEvent) | Byte 0: CP state enum (0=A..5=F, 6=PowerOn, 7=PowerOff) | On state change |
| `0x211` | `capabilities` (HardwareCapabilities) | Full capability struct (8 bytes) | Once at startup |
| `0x212` | `telemetry` (Telemetry) | Temps, fan, supply voltages, relay state | Periodic (1 s) |
| `0x213` | `ac_pp_ampacity` | Byte 0: PP code enum | On connector insert |
| `0x214` | `request_stop_transaction` | Byte 0: reason enum | On physical stop button |

### 3.3 isolation_monitor Interface (CM5 ↔ STM32)

| Direction | EVerest Command/Var | CAN ID | Payload |
|-----------|-------------------|--------|---------|
| CM5 → STM32 | `start()` | `0x120` | Byte 0: 1 (start continuous) |
| CM5 → STM32 | `stop()` | `0x120` | Byte 0: 0 (stop) |
| CM5 → STM32 | `start_self_test(V)` | `0x121` | Bytes 0-1: test voltage (uint16, LE) |
| STM32 → CM5 | `isolation_measurement` | `0x220` | Bytes 0-3: resistance (uint32, ohms); Bytes 4-5: voltage (uint16) |
| STM32 → CM5 | `self_test_result` | `0x221` | Byte 0: 0=fail, 1=pass |

### 3.4 ac_rcd Interface (STM32 → CM5)

| Direction | EVerest Command/Var | CAN ID | Payload |
|-----------|-------------------|--------|---------|
| CM5 → STM32 | `self_test()` | `0x130` | Byte 0: 1 |
| CM5 → STM32 | `reset()` | `0x130` | Byte 0: 2 |
| STM32 → CM5 | `rcd_current_mA` | `0x230` | Bytes 0-1: current (uint16, 0.1 mA units) |

> [!note] RCD trip
> The RCD trip itself is hardwired in the interlock chain and does not depend on CAN communication. The `rcd_current_mA` variable is telemetry only. If the RCD trips, the safety supervisor reports it as a fault via CAN `0x202` (Fault Detail), and the BSP module raises an `evse_board_support/MREC2GroundFailure` error in EVerest.

### 3.5 Error Reporting (STM32 → CM5)

The safety supervisor's fault codes map to EVerest's typed error system:

| STM32 Fault | CAN `0x202` Code | EVerest Error | Interface |
|-------------|-------------------|---------------|-----------|
| F01: DC overvoltage | `0x01` | `evse_board_support/PermanentFault` | board_support |
| F02: DC overcurrent | `0x02` | `evse_board_support/PermanentFault` | board_support |
| F03: IMD fault | `0x03` | `isolation_monitor/DeviceFault` | imd |
| F04: RCD trip | `0x04` | `evse_board_support/MREC2GroundFailure` | board_support |
| F05: Contactor weld | `0x05` | `evse_board_support/PermanentFault` | board_support |
| F06: Precharge timeout | `0x06` | `evse_board_support/PermanentFault` | board_support |
| F07: CM5 watchdog | — | _(CM5 is down, no one to report to)_ | — |
| F08: Over-temperature | `0x08` | `evse_board_support/PermanentFault` | board_support |
| F09: Door interlock | `0x09` | `evse_board_support/EnclosureOpen` | board_support |
| F10: Contactor mismatch | `0x0A` | `evse_board_support/PermanentFault` | board_support |
| F11: CAN loss | `0x0B` | `evse_board_support/CommunicationFault` | board_support |
| F12: ADC failure | `0x0C` | `evse_board_support/PermanentFault` | board_support |

When the BSP module receives a fault frame on CAN `0x202`, it calls `raise_error()` on the appropriate interface. EvseManager then transitions to an error state and may report to the backend via OCPP `StatusNotification`.

## 4. BSP Module Implementation

### 4.1 Module Manifest

```yaml
# modules/SafetySupervisorBSP/manifest.yaml
description: >
  Board support driver for STM32-based safety supervisor
  connected via CAN bus. Provides BSP, IMD, RCD, and
  connector lock interfaces to EvseManager.
config:
  can_device:
    description: Linux SocketCAN interface name
    type: string
    default: can0
  node_id:
    description: CAN node ID of the safety supervisor
    type: integer
    default: 1
  heartbeat_interval_ms:
    description: Heartbeat send interval to safety supervisor
    type: integer
    default: 500
  heartbeat_timeout_ms:
    description: Max time without response before raising CommunicationFault
    type: integer
    default: 2000
provides:
  board_support:
    interface: evse_board_support
    description: CP signaling, contactor control, telemetry
  imd:
    interface: isolation_monitor
    description: Insulation monitoring via safety supervisor
  rcd:
    interface: ac_rcd
    description: Residual current monitoring
  connector_lock:
    interface: connector_lock
    description: Connector latch motor control
```

### 4.2 Module Structure

```
modules/SafetySupervisorBSP/
├── manifest.yaml
├── CMakeLists.txt
├── SafetySupervisorBSP.hpp            # Module class
├── SafetySupervisorBSP.cpp            # CAN init, heartbeat thread
├── can_protocol.hpp                   # CAN ID definitions, frame encoding
├── board_support/
│   └── evse_board_supportImpl.cpp     # Command handlers + event publishing
├── imd/
│   └── isolation_monitorImpl.cpp      # IMD command handlers
├── rcd/
│   └── ac_rcdImpl.cpp                 # RCD handlers
└── connector_lock/
    └── connector_lockImpl.cpp         # Lock/unlock handlers
```

### 4.3 Core Module Class

```cpp
// SafetySupervisorBSP.hpp
class SafetySupervisorBSP : public Everest::ModuleBase {
public:
    // CAN communication layer
    CanDevice can;

    // Shared state (written by CAN RX thread, read by impl handlers)
    std::atomic<uint8_t> current_cp_state{0};   // A=0..F=5
    std::atomic<bool> relay_closed{false};
    std::atomic<uint32_t> imd_resistance_ohm{0};
    std::atomic<uint16_t> dc_bus_voltage{0};
    std::atomic<uint16_t> rcd_current_01mA{0};
    std::atomic<uint16_t> last_fault_code{0};

    // Heartbeat management
    std::thread heartbeat_thread;
    std::atomic<bool> supervisor_alive{false};
    std::chrono::steady_clock::time_point last_rx_time;

    void init() override;
    void ready() override;

private:
    void heartbeat_loop();
    void handle_can_rx(uint32_t can_id, const std::vector<uint8_t>& data);
};
```

### 4.4 Initialization and CAN RX

```cpp
// SafetySupervisorBSP.cpp
void SafetySupervisorBSP::init() {
    can.open_device(config.can_device.c_str());

    can.set_rx_callback([this](uint32_t id, const auto& data) {
        handle_can_rx(id, data);
    });
}

void SafetySupervisorBSP::ready() {
    // Start heartbeat thread after all modules are initialized
    heartbeat_thread = std::thread(&SafetySupervisorBSP::heartbeat_loop, this);
}

void SafetySupervisorBSP::handle_can_rx(uint32_t id, const auto& data) {
    last_rx_time = std::chrono::steady_clock::now();

    switch (id) {
        case 0x200:  // Safety Status (periodic)
            current_cp_state = data[0] & 0x0F;
            // Update fault bitmask, contactor status
            break;

        case 0x210:  // CP State Change Event
            p_board_support->publish_event(map_to_bsp_event(data[0]));
            break;

        case 0x220:  // Isolation Measurement
            imd_resistance_ohm = unpack_u32_le(data, 0);
            dc_bus_voltage = unpack_u16_le(data, 4);
            p_imd->publish_isolation_measurement({
                .resistance_F_Ohm = imd_resistance_ohm.load(),
                .voltage_V = dc_bus_voltage.load()
            });
            break;

        case 0x221:  // IMD Self-Test Result
            p_imd->publish_self_test_result(data[0] == 1);
            break;

        case 0x230:  // RCD Current
            rcd_current_01mA = unpack_u16_le(data, 0);
            p_rcd->publish_rcd_current_mA(rcd_current_01mA / 10.0);
            break;

        case 0x202:  // Fault Detail
            handle_fault_frame(data);
            break;
    }
}

void SafetySupervisorBSP::heartbeat_loop() {
    while (!should_exit) {
        // Send heartbeat (CAN 0x100)
        uint8_t counter = heartbeat_counter++;
        can.tx(0x100, {counter, 0x01});  // 0x01 = running

        std::this_thread::sleep_for(
            std::chrono::milliseconds(config.heartbeat_interval_ms));

        // Check if supervisor is responding
        auto elapsed = std::chrono::steady_clock::now() - last_rx_time;
        if (elapsed > std::chrono::milliseconds(config.heartbeat_timeout_ms)) {
            if (supervisor_alive.exchange(false)) {
                p_board_support->raise_error(
                    "evse_board_support/CommunicationFault",
                    "Safety supervisor CAN heartbeat lost");
            }
        } else {
            if (!supervisor_alive.exchange(true)) {
                p_board_support->clear_error(
                    "evse_board_support/CommunicationFault");
            }
        }
    }
}
```

### 4.5 BSP Command Handlers

```cpp
// board_support/evse_board_supportImpl.cpp

void evse_board_supportImpl::handle_enable(bool& value) {
    mod->can.tx(0x110, {static_cast<uint8_t>(value ? 1 : 0)});
}

void evse_board_supportImpl::handle_pwm_on(double& value) {
    uint16_t duty = static_cast<uint16_t>(value * 100);  // % → 0.01% units
    mod->can.tx(0x111, {
        static_cast<uint8_t>(duty & 0xFF),
        static_cast<uint8_t>(duty >> 8)
    });
}

void evse_board_supportImpl::handle_cp_state_X1() {
    mod->can.tx(0x111, {0x11, 0x27});  // 10001 in LE
}

void evse_board_supportImpl::handle_cp_state_F() {
    mod->can.tx(0x111, {0x00, 0x00});
}

void evse_board_supportImpl::handle_allow_power_on(
        types::evse_board_support::PowerOnOff& value) {
    uint8_t reason = 0;
    if (value.reason == "DCCableCheck")        reason = 1;
    else if (value.reason == "DCPreCharge")    reason = 2;
    else if (value.reason == "FullPowerCharging") reason = 3;
    else if (value.reason == "PowerOff")       reason = 4;

    mod->can.tx(0x101, {
        static_cast<uint8_t>(value.allow_power_on ? 1 : 0),
        reason
    });
}
```

## 5. YAML Configuration

### 5.1 Complete DC Charger Config

```yaml
active_modules:

  # ── Safety Supervisor BSP (provides BSP + IMD + RCD + Lock) ──
  safety_bsp:
    module: SafetySupervisorBSP
    config_module:
      can_device: can0
      node_id: 1
      heartbeat_interval_ms: 500
      heartbeat_timeout_ms: 2000

  # ── DC Power Supply (CAN #1, separate bus) ──
  powersupply_dc:
    module: DCPowerSupplyDriver        # Custom or InfyPower module
    config_module:
      can_device: can1                 # CAN #1 for power bricks

  # ── Over-Voltage Monitor ──
  ovm:
    module: OverVoltageMonitor
    connections:
      bsp:
        - module_id: safety_bsp
          implementation_id: board_support

  # ── SLAC + ISO 15118 ──
  slac:
    module: EvseSlac
    config_implementation:
      main:
        device: eth_plc

  iso15118:
    module: EvseV2G
    config_module:
      device: eth_plc
    connections:
      security:
        - module_id: evse_security
          implementation_id: main

  # ── Core EVSE Manager ──
  evse_manager:
    module: EvseManager
    config_module:
      connector_id: 1
      charge_mode: DC
      ac_hlc_enabled: true
      ac_hlc_use_5percent: true
      session_logging: true
      cable_check_wait_number_of_imd_measurements: 3
    connections:
      bsp:
        - module_id: safety_bsp
          implementation_id: board_support
      imd:
        - module_id: safety_bsp
          implementation_id: imd
      ac_rcd:
        - module_id: safety_bsp
          implementation_id: rcd
      connector_lock:
        - module_id: safety_bsp
          implementation_id: connector_lock
      powersupply_DC:
        - module_id: powersupply_dc
          implementation_id: main
      slac:
        - module_id: slac
          implementation_id: main
      hlc:
        - module_id: iso15118
          implementation_id: charger
      over_voltage_monitor:
        - module_id: ovm
          implementation_id: main

  # ── Auth + OCPP ──
  auth:
    module: Auth
    config_module:
      selection_algorithm: FindFirst
    connections:
      token_provider:
        - module_id: rfid
          implementation_id: main
      token_validator:
        - module_id: ocpp
          implementation_id: auth_validator
      evse_manager:
        - module_id: evse_manager
          implementation_id: evse

  ocpp:
    module: OCPP201
    connections:
      evse_manager:
        - module_id: evse_manager
          implementation_id: evse
      security:
        - module_id: evse_security
          implementation_id: main

  energy_manager:
    module: EnergyManager
    connections:
      energy_trunk:
        - module_id: evse_manager
          implementation_id: energy_grid
```

### 5.2 Wiring Summary

```
                                YAML Connection Map

  EvseManager.bsp ──────────────────► safety_bsp.board_support
  EvseManager.imd ──────────────────► safety_bsp.imd
  EvseManager.ac_rcd ───────────────► safety_bsp.rcd
  EvseManager.connector_lock ───────► safety_bsp.connector_lock
  EvseManager.powersupply_DC ───────► powersupply_dc.main
  EvseManager.slac ─────────────────► slac.main
  EvseManager.hlc ──────────────────► iso15118.charger
  EvseManager.over_voltage_monitor ─► ovm.main
  Auth.evse_manager ────────────────► evse_manager.evse
  OCPP.evse_manager ────────────────► evse_manager.evse
  EnergyManager.energy_trunk ───────► evse_manager.energy_grid
```

## 6. DC Charging Sequence: Integration View

This traces a complete DC charging session through both the EVerest modules and the safety supervisor.

```
Phase           EVerest (CM5)                    CAN Bus         Safety Supervisor (STM32)
─────────────── ──────────────────────────────── ─────────────── ───────────────────────────

1. EV Plugs In                                                   CP: A → B detected
                                                 ◀── 0x210 ───  Publish CP state B
                EvseManager: State A → B
                connector_lock.lock()
                                                 ─── 0x113 ──▶  Lock motor engaged
                bsp.pwm_on(5%)
                                                 ─── 0x111 ──▶  Generate 5% PWM on CP

2. SLAC/ISO     SLAC module handles matching
   15118        ISO 15118: SessionSetup,
                ChargeParameterDiscovery

3. CableCheck   imd.start_self_test(500V)
                                                 ─── 0x121 ──▶  Begin IMD self-test
                bsp.allow_power_on(DCCableCheck)
                                                 ─── 0x101 ──▶  Validate → AC_CLOSE →
                                                                 PRECHARGE → DC_CLOSE
                                                                 (at low voltage)
                                                 ◀── 0x210 ───  PowerOn event
                                                 ◀── 0x221 ───  self_test_result = true
                EvseManager: CableCheck pass

4. PreCharge    powersupply_DC.setMode(Precharge)
                powersupply_DC.setExportV(target, 2A)
                bsp.allow_power_on(DCPreCharge)
                                                 ─── 0x101 ──▶  Already closed, update reason
                Monitor voltage convergence
                via imd.isolation_measurement
                                                 ◀── 0x220 ───  Resistance + voltage readings

5. Charging     bsp.allow_power_on(FullPowerCharging)
                                                 ─── 0x101 ──▶  Update reason → CHARGING
                imd.start()
                                                 ─── 0x120 ──▶  Continuous IMD monitoring
                powersupply_DC.setExportV(V, I)
                EnergyManager provides limits
                                                 ◀── 0x220 ───  Periodic isolation readings
                                                 ◀── 0x212 ───  Periodic telemetry

6. Session End  powersupply_DC.setMode(Off)
                (blocks until I < threshold)
                bsp.allow_power_on(PowerOff)
                                                 ─── 0x101 ──▶  SHUTDOWN sequence:
                                                                 disable bricks → open DC →
                                                                 open AC
                                                 ◀── 0x210 ───  PowerOff event
                imd.stop()
                                                 ─── 0x120 ──▶  Stop IMD
                connector_lock.unlock()
                                                 ─── 0x113 ──▶  Unlock motor
                                                                 CP: B → A (EV unplugs)
                                                 ◀── 0x210 ───  CP state A
                EvseManager: State A (idle)

7. Fault        (at any point)
   (example:                                     ◀── 0x202 ───  F03: IMD fault
    IMD fault)                                   ◀── 0x210 ───  PowerOff (contactors opened)
                BSP raises isolation_monitor/
                  DeviceFault
                EvseManager → error state
                OCPP: StatusNotification(Faulted)
```

## 7. Dual Watchdog Strategy

The safety supervisor and CM5 watch each other. Failure of either side leads to a safe shutdown.

```
┌──────────────────────────────────────────────────────────────────┐
│                     WATCHDOG ARCHITECTURE                         │
│                                                                  │
│  CM5 → STM32 Watchdog (CAN 0x100):                              │
│    CM5 sends heartbeat every 500 ms                              │
│    STM32 expects it within 2 s                                   │
│    Timeout → F07 → FAULT state → contactors open                 │
│                                                                  │
│  STM32 → CM5 Watchdog (CAN 0x200):                              │
│    STM32 sends Safety Status every 100 ms                        │
│    BSP module expects response within 2 s                        │
│    Timeout → raise CommunicationFault                            │
│    EvseManager → blocks new sessions, terminates active session  │
│                                                                  │
│  STM32 Internal (IWDG):                                          │
│    Independent watchdog timer, 500 ms window                     │
│    If firmware hangs → hardware reset → INIT → all outputs OFF   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 8. Safety Boundary: What Each Side Must Never Do

This table clarifies the hard boundary between EVerest and the safety supervisor.

| Rule | Rationale |
|------|-----------|
| **EVerest must never directly drive contactors** | Non-real-time Linux cannot guarantee response times; a hung process must not leave contactors closed |
| **EVerest must never bypass the safety supervisor** | All power path commands go through `allow_power_on`, which the supervisor can reject |
| **Safety supervisor must never run session logic** | Session state, auth, billing, and protocol handling belong to EVerest; the supervisor only validates physical preconditions |
| **Safety supervisor must never depend on CAN for shutdown** | Emergency stop and hardware faults use the hardwired interlock chain (Layer 1), not CAN messages |
| **IMD/RCD trip relays must be hardwired in series with contactors** | Software monitoring is supplementary; the physical failsafe is independent of both CM5 and STM32 |

## 9. Testing Strategy

### 9.1 Hardware-in-the-Loop (HIL)

| Test | Method | Pass Criteria |
|------|--------|---------------|
| CP state transitions | Inject resistor network on CP line | BSP publishes correct A-F events |
| Contactor sequencing | Monitor contactor coils via scope | Correct sequence and timing |
| IMD self-test | Inject known resistance to IMD | `self_test_result = true` propagated |
| Watchdog (CM5 side) | Kill EVerest process | STM32 enters FAULT within 2 s |
| Watchdog (STM32 side) | Disconnect CAN from STM32 | BSP raises CommunicationFault |
| E-STOP | Press E-STOP during charging | All contactors open within 1 ms |
| Fault propagation | Trigger IMD fault | EvseManager enters error, OCPP reports |

### 9.2 Software-in-the-Loop (SIL)

EVerest includes a simulation framework. A `SafetySupervisorSimulator` module can be created to test session flows without hardware:

```yaml
# config-sil-safety-supervisor.yaml
active_modules:
  safety_bsp_sim:
    module: SafetySupervisorSimulator
    config_module:
      initial_cp_state: A
      imd_resistance_ohm: 10000000   # 10 MΩ (healthy)
      simulate_faults: false

  evse_manager:
    module: EvseManager
    config_module:
      charge_mode: DC
    connections:
      bsp:
        - module_id: safety_bsp_sim
          implementation_id: board_support
      imd:
        - module_id: safety_bsp_sim
          implementation_id: imd
```

## 10. Related Documentation

- [[03 - Safety Supervisor Controller]] — Safety supervisor firmware, state machine, CAN protocol
- [[01 - Software Framework]] — EVerest framework overview and module types
- [[02 - Communication Protocols]] — CAN bus topology and wiring
- [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]] — EVerest interface specifications and driver patterns
- [[research/01 - Safety Philosophy|01 - Safety Philosophy]] — Hardware interlock chain design
- [[research/02 - CM5 based Main Controller|02 - CM5 based Main Controller]] — CM5 hardware and EVSE aux board
