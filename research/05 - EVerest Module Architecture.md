---
tags: [dcfc, everest, software, bsp, hardware-integration, can, serial, interfaces]
created: 2026-02-26
---

# EVerest Module Architecture: Hardware Integration Deep Dive

> [!note] Source
> Research from EVerest official docs (everest.github.io/nightly), GitHub source code (EVerest/everest-core), and generated reference documentation. All interface YAMLs verified against the main branch.

## 1. Runtime Architecture Overview

EVerest is a microservice framework for EV chargers running on Linux. Three core processes coordinate all activity:

- **Manager process** — orchestrates module lifecycle, reads the YAML config, spawns module processes
- **Module instances** — separate OS processes, each running one compiled module. Multiple instances of the same module can coexist with different IDs and configs
- **Mosquitto MQTT broker** — the IPC backbone between all modules

### MQTT Topic Pattern

Every interface communication follows a deterministic topic scheme:

```
everest/{module_instance_id}/{implementation_id}/cmd     # synchronous commands
everest/{module_instance_id}/{implementation_id}/var     # asynchronous variable publications
```

For the external API layer (used when bridging to external microcontrollers over MQTT without writing a native C++ module):

```
everest_api/1/{api_type}/{module_id}/e2m/{message}   # EVerest → external (commands going out)
everest_api/1/{api_type}/{module_id}/m2e/{message}   # external → EVerest (vars coming in)
```

---

## 2. The Interface Contract System

EVerest modules communicate exclusively through **declared interfaces**. Each interface is a YAML file defining:

- **cmds** — synchronous RPC calls with typed arguments and return values
- **vars** — asynchronous published variables (pub/sub)
- **errors** — typed error codes that can be raised or cleared

A module's `manifest.yaml` declares:
- `provides:` — named implementations of interfaces this module offers
- `requires:` — interface implementations this module needs from other modules

The YAML run config wires these together: a `requires` slot in one module is connected to a `provides` slot in another using `module_id` + `implementation_id`.

---

## 3. Safety-Critical Interfaces: Full Specification

### 3.1 `evse_board_support` Interface

**Source:** `interfaces/evse_board_support.yaml`

The central hardware abstraction. Every EVSE connector needs exactly one provider of this interface. It covers Control Pilot (CP) signaling, output contactors (relays), and hardware capability reporting.

#### Commands (EVerest → BSP driver)

| Command | Arguments | Description |
|---|---|---|
| `enable` | `value: boolean` | Enable/disable the charging port. When disabled, BSP prevents new sessions and caches the last PWM value for re-application on re-enable |
| `pwm_on` | `value: number` (0–100%) | Set PWM duty cycle. Encodes the current limit per IEC 61851-1. Only active when port is enabled |
| `cp_state_X1` | — | Set CP to constant high voltage (no PWM, State X1 = standby ready) |
| `cp_state_F` | — | Set CP to continuous −12V (State F = EVSE not available / error) |
| `cp_state_E` | — | Set CP to 0V (State E = simulated disconnection; requires hardware support) |
| `allow_power_on` | `value: PowerOnOff` | Master relay control. If `false`, relays must never close. The `reason` field tells the BSP the charging phase (DCCableCheck, DCPreCharge, FullPowerCharging, PowerOff) |
| `ac_switch_three_phases_while_charging` | `value: boolean` | Switch 1↔3 phase mid-session (optional, hardware-specific sequence required) |
| `evse_replug` | `value: integer` (ms) | Simulate replug for testing (optional, do not implement in production) |
| `ac_set_overcurrent_limit_A` | `value: number` (A) | Set hardware overcurrent trip threshold (ignore if hardware lacks this) |

#### Variables (BSP driver → EVerest, published asynchronously)

| Variable | Type | Description |
|---|---|---|
| `event` | `BspEvent` | CP state transitions and relay state changes — the primary event stream |
| `capabilities` | `HardwareCapabilities` | Hardware current/phase limits. Must publish at least once on startup |
| `ac_nr_of_phases_available` | `integer` (1–3) | Instantaneous phases available |
| `telemetry` | `Telemetry` | Temperature, fan RPM, supply voltages, relay state |
| `ac_pp_ampacity` | `ProximityPilot` | PP cable ampacity (only for Type 2 socket, not fixed cable or DC) |
| `request_stop_transaction` | `StopTransactionRequest` | BSP can request a graceful stop (e.g. physical stop button) |

#### Key Types

**`BspEvent.event`** (enum):
```
A, B, C, D, E, F       # IEC 61851-1 CP states
PowerOn, PowerOff       # Relay closure/opening confirmation from hardware
EvseReplugStarted, EvseReplugFinished
Disconnected            # EV-side only
```

**`HardwareCapabilities`** (required fields):
```yaml
max_current_A_import / min_current_A_import
max_phase_count_import / min_phase_count_import
max_current_A_export / min_current_A_export     # for V2G/BPT
max_phase_count_export / min_phase_count_export
supports_changing_phases_during_charging: bool
supports_cp_state_E: bool
connector_type: IEC62196Type2Cable | IEC62196Type2Socket
```

**`PowerOnOff`**:
```yaml
allow_power_on: bool
reason: DCCableCheck | DCPreCharge | FullPowerCharging | PowerOff
```

**`Telemetry`**:
```yaml
evse_temperature_C: number
plug_temperature_C: number   # optional
fan_rpm: number
supply_voltage_12V: number
supply_voltage_minus_12V: number
relais_on: bool
```

#### Errors (raised via EVerest error framework)

`evse_board_support/DiodeFault`, `VentilationNotAvailable`, `BrownOut`, `EnergyManagement`, `PermanentFault`, `MREC2GroundFailure` through `MREC26CutCable`, `TiltDetected`, `WaterIngressDetected`, `EnclosureOpen`, `VendorError`, `VendorWarning`, `CommunicationFault`

---

### 3.2 `isolation_monitor` Interface

**Source:** `interfaces/isolation_monitor.yaml`

For DC charging only. Implements IEC 61557-8 isolation monitoring between DC+/DC− and protective earth.

#### Commands

| Command | Arguments | Description |
|---|---|---|
| `start` | — | Begin continuous isolation measurement. Publish results at regular intervals until stopped |
| `stop` | — | Stop measurement and publishing |
| `start_self_test` | `test_voltage_V: number` | Trigger IMD self-test during CableCheck phase (DC voltage will be present). Returns immediately; result published via `self_test_result` var. Can take up to 20 seconds on some hardware |

#### Variables

| Variable | Type | Description |
|---|---|---|
| `isolation_measurement` | `IsolationMeasurement` | Periodic resistance measurement |
| `self_test_result` | `boolean` | True = pass, False = fail. Published once after `start_self_test` |

**`IsolationMeasurement`**:
```yaml
resistance_F_Ohm: number   # required — isolation resistance to PE in ohms
voltage_V: number           # optional — DC bus voltage
voltage_to_earth_l1e_V: number   # optional
voltage_to_earth_l2e_V: number   # optional
```

#### Errors

`isolation_monitor/DeviceFault`, `CommunicationFault`, `VendorError`, `VendorWarning`

> [!tip] Integration note
> EvseManager calls `start_self_test` during CableCheck. It waits for `self_test_result` before proceeding to PreCharge. The config `cable_check_wait_number_of_imd_measurements` controls how many good readings are required.

---

### 3.3 `ac_rcd` Interface

**Source:** `interfaces/ac_rcd.yaml`

AC Residual Current Device. The actual emergency trip is wired directly in hardware (not software-controlled). This interface provides monitoring and test capability only.

#### Commands

| Command | Result | Description |
|---|---|---|
| `self_test` | void | Trigger RCD self-test. Raises `Selftest` error on failure |
| `reset` | `boolean` | Attempt reset after a trip. True = success. May not be supported on all hardware |

#### Variables

| Variable | Type | Description |
|---|---|---|
| `rcd_current_mA` | `number` | Residual current in mA — reporting only, does not trigger any action |

#### Errors

`ac_rcd/Selftest`, `DC`, `AC`, `MREC2GroundFailure`, `VendorError`, `VendorWarning`

> [!warning]
> Note that `ac_rcd` errors are **also** part of the `evse_board_support` error reference in the interface YAML. In the YetiDriver implementation, RCD errors from the MCU arrive on the serial link and are routed appropriately. If using the `evse_board_support_API` bridge module, RCD errors are submitted via the `raise_error` MQTT topic.

---

### 3.4 `connector_lock` Interface

**Source:** `interfaces/connector_lock.yaml`

Controls a motorized connector latch (Type 2 sockets, CCS inlets with mechanical lock).

#### Commands

| Command | Description |
|---|---|
| `lock` | Lock the connector |
| `unlock` | Unlock the connector (normal session end or OCPP-forced) |

No variables. No arguments. All feedback via errors.

#### Errors

`connector_lock/ConnectorLockCapNotCharged`, `ConnectorLockUnexpectedOpen`, `ConnectorLockUnexpectedClose`, `ConnectorLockFailedLock`, `ConnectorLockFailedUnlock`, `MREC1ConnectorLockFailure`, `VendorError`, `VendorWarning`

---

### 3.5 `evse_manager` Interface

**Source:** `interfaces/evse_manager.yaml`

The high-level session management interface exposed by EvseManager to OCPP, Auth, and the API layer.

#### Commands (called by external modules on EvseManager)

| Command | Key Arguments | Description |
|---|---|---|
| `get_evse` | — | Returns EVSE info including connectors |
| `enable_disable` | `connector_id`, `cmd_source` | Enable/disable power path |
| `authorize_response` | `provided_token`, `validation_result` | Auth module delivers token validation result |
| `withdraw_authorization` | — | Revoke auth (timeout, disconnect) |
| `reserve` | `reservation_id` | Mark EVSE reserved |
| `cancel_reservation` | — | Clear reservation |
| `pause_charging` | — | Request pause |
| `resume_charging` | — | Resume from pause |
| `stop_transaction` | `request: StopTransactionRequest` | End session |
| `force_unlock` | `connector_id` | Emergency connector release |
| `external_ready_to_start_charging` | — | Signal completion of external init (used with `waiting_for_external_ready`) |

#### Variables (published by EvseManager)

| Variable | Type | Description |
|---|---|---|
| `session_event` | `SessionEvent` | All session state transitions |
| `limits` | `Limits` | Current charging limits |
| `ev_info` | `EVInfo` | EV details if available via HLC |
| `hw_capabilities` | `HardwareCapabilities` | Republished from BSP |
| `powermeter` | `Powermeter` | From billing-relevant meter |
| `ready` | `boolean` | EvseManager ready to accept charging |
| `waiting_for_external_ready` | `boolean` | Signals external init delay |
| `selected_protocol` | `string` | Active charging protocol |

#### Errors raised by EvseManager

`evse_manager/Internal`, `MREC4OverCurrentFailure`, `MREC5OverVoltage`, `MREC9AuthorizationTimeout`, `PowermeterTransactionStartFailed`, `Inoperative`, `MREC22ResistanceFault`, `MREC11CableCheckFault`, `VoltagePlausibilityFault`

---

## 4. EvseManager: Coordination Logic

EvseManager is the central state machine, defined in `modules/EVSE/EvseManager/`. It:

1. **Subscribes to BSP `event` var** — drives the IEC 61851 state machine (A→B→C→D→E→F transitions). CP state changes from the BSP hardware trigger session start/stop, power on/off, etc.

2. **Controls CP PWM** — calls `bsp.pwm_on(duty_cycle)` to encode current limits. For HLC (ISO 15118), switches to 5% PWM to signal digital communication capability.

3. **Controls contactors** — calls `bsp.allow_power_on(PowerOnOff)` with the appropriate `reason`:
   - `DCCableCheck` — close relay at low voltage to allow IMD self-test
   - `DCPreCharge` — hold relay closed during voltage ramp-up to match battery voltage
   - `FullPowerCharging` — normal charge contactor state
   - `PowerOff` — open relays. BSP must ensure power supply current is below switching threshold first.

4. **Calls IMD** — on DC sessions: calls `imd.start()`, then `imd.start_self_test(voltage)` during CableCheck. Monitors `isolation_measurement` during charging. Raises `MREC22ResistanceFault` if isolation drops below threshold.

5. **Calls connector_lock** — locks on plug-in (State B), unlocks on end-of-session.

6. **Error propagation** — subscribes to errors from all required interfaces. Most errors → `Inoperative` state → session blocked. `Severity::High` errors trigger immediate power cut; `Medium/Low` maintain PWM during HLC to report fault to EV via ISO 15118.

7. **Energy management** — never self-assigns limits. Requests energy schedules from EnergyManager (which sets `enforce_limits` on the module). The power supply is controlled via `powersupply_DC.setMode()` and `setExportVoltageCurrent()`.

### EvseManager Required Connections (manifest.yaml)

```yaml
requires:
  bsp:                   # REQUIRED — evse_board_support (1 connection)
    interface: evse_board_support
  ac_rcd:                # optional (0–1)
    interface: ac_rcd
  connector_lock:        # optional (0–1)
    interface: connector_lock
  powermeter_grid_side:  # optional (0–1) — billing fallback
    interface: powermeter
  powermeter_car_side:   # optional (0–1) — preferred billing meter
    interface: powermeter
  slac:                  # optional (0–1)
    interface: slac
  hlc:                   # optional (0–1) — ISO 15118 charger stack
    interface: ISO15118_charger
  imd:                   # optional (0–1) — required for DC
    interface: isolation_monitor
  over_voltage_monitor:  # optional (0–1) — required for DC (IEC 61851-23:2023)
    interface: over_voltage_monitor
  powersupply_DC:        # optional (0–1) — required for DC
    interface: power_supply_DC
  store:                 # optional (0–1)
    interface: kvs
```

---

## 5. Hardware Driver Module Patterns

### Pattern A: Native C++ Module over Serial (COBS + Protobuf)

The **YetiDriver** and **TIDA010939** BSP drivers both use:

**Transport:** 3.3V TTL UART (default 115200 baud 8N1)
**Framing:** COBS (Consistent Overhead Byte Stuffing) — each packet is a self-delimiting byte-stuffed frame, no special start/end bytes needed
**Serialization:** Protocol Buffers (nanopb for embedded side)
**GPIO:** One reset GPIO, one wakeup GPIO (optional)

**YetiDriver proto message structure** (`yeti.proto`):

```protobuf
// EVerest → MCU
message EverestToMcu {
  oneof payload {
    bool connector_lock = 102;      // false = unlock, true = lock
    uint32 pwm_duty_cycle = 103;    // 0.01% units; 0 = State F; 10000 = X1
    bool allow_power_on = 104;
    bool reset = 105;
    bool set_number_of_phases = 106;
    KeepAlive keep_alive = 100;
    FirmwareUpdate firmware_update = 16;
  }
}

// MCU → EVerest
message McuToEverest {
  oneof payload {
    KeepAliveLo keep_alive = 3;     // includes HW caps, firmware version
    ResetReason reset = 101;
    CpState cp_state = 102;         // STATE_A through STATE_F
    bool relais_state = 103;        // relay feedback
    ErrorFlags error_flags = 104;
    Telemetry telemetry = 105;
    PpState pp_state = 106;         // PP cable ampacity
    LockState lock_state = 107;
    PowerMeter power_meter = 108;
  }
}

enum CpState { STATE_A=0; STATE_B=1; STATE_C=2; STATE_D=3; STATE_E=4; STATE_F=5; }

message ErrorFlags {
  bool diode_fault = 1;
  bool rcd_selftest_failed = 2;
  bool rcd_triggered = 3;
  bool ventilation_not_available = 4;
  bool connector_lock_failed = 5;
  bool cp_signal_fault = 6;
  bool over_current = 7;
}

message KeepAliveLo {
  uint32 time_stamp = 1;
  uint32 hw_type = 2;
  uint32 hw_revision = 3;
  uint32 protocol_version_major = 4;
  uint32 protocol_version_minor = 5;
  string sw_version_string = 6;
  float hwcap_max_current = 7;
  float hwcap_min_current = 8;
  uint32 hwcap_max_phase_count = 9;
  uint32 hwcap_min_phase_count = 10;
  bool supports_changing_phases_during_charging = 11;
}
```

**BSP implementation pattern** (how the YetiDriver maps serial signals to EVerest interface):

```cpp
// In evse_board_supportImpl::init():

// CP state from MCU → publish BspEvent to EVerest
mod->serial.signalCPState.connect([this](CpState cp_state) {
    auto event = cast_event_type(cp_state);
    publish_event(event);         // → everest/{id}/board_support/var/event
});

// Relay state confirmation from MCU → PowerOn/PowerOff event
mod->serial.signalRelaisState.connect([this](bool relais_state) {
    publish_event(cast_event_type(relais_state));
});

// Hardware capabilities from MCU KeepAlive → publish to EVerest
mod->serial.signalKeepAliveLo.connect([this](KeepAliveLo l) {
    caps.max_current_A_import = l.hwcap_max_current;
    publish_capabilities(caps);   // → everest/{id}/board_support/var/capabilities
});

// Command handlers (EVerest → serial):
void handle_pwm_on(double& value) {
    mod->serial.setPWM(value * 100);  // convert % to 0.01% units
}
void handle_allow_power_on(PowerOnOff& value) {
    mod->serial.allowPowerOn(value.allow_power_on);
}
void handle_cp_state_X1() { mod->serial.setPWM(10001); }  // sentinel value
void handle_cp_state_F()  { mod->serial.setPWM(0); }
```

---

### Pattern B: Native C++ Module over CAN (SocketCAN)

The **InfyPower_BEG1K075G** DC power supply driver uses Linux SocketCAN:

```cpp
// CanDevice.hpp — base class for CAN communication
class CanDevice {
public:
    bool open_device(const char* dev);   // opens /dev/can0 via SocketCAN
protected:
    virtual void rx_handler(uint32_t can_id, const std::vector<uint8_t>& payload);
    bool _tx(uint32_t can_id, const std::vector<uint8_t>& payload);
private:
    int can_fd;             // SocketCAN file descriptor
    std::thread rx_thread_handle;
};
```

CAN IDs encode addressing and commands in 29-bit extended frames:
- Bits 0–7: Source address
- Bits 8–14: Destination address (0x3F = broadcast, 0xF0 = controller default)
- Bits 15–21: Command number (0x23 = read, 0x24 = write)
- Bits 22–25: Device number (0x0A = single module, 0x0B = group)
- Bits 26–28: Error code

Module manifest `config`: `can_device: can0` (Linux CAN interface name)

The `InfyPower_BEG1K075G` module provides the `power_supply_DC` interface and uses `open_device(config.can_device.c_str())` in `init()`.

---

### Pattern C: Modbus RTU via SerialCommHub

For isolation monitors (Bender isoCHA425HV, Dold RN5893) and power meters:

The **SerialCommHub** module provides a `serial_communication_hub` interface. Multiple driver modules share one physical RS-485 bus by all connecting to the same SerialCommHub instance.

**Bender isoCHA425HV** configuration:
```yaml
iso_monitor:
  module: Bender_isoCHA425HV
  config_implementation:
    main:
      imd_device_id: 3          # Modbus device address
  connections:
    serial_comm_hub:
      - implementation_id: main
        module_id: comm_hub

comm_hub:
  module: SerialCommHub
  config_implementation:
    main:
      serial_port: /dev/cb_rs485
      baudrate: 19200
      parity: 2                 # even parity per Bender default
```

---

### Pattern D: External Process via `evse_board_support_API` (MQTT Bridge)

When the safety controller is a separate system (e.g., a different OS process, a Python script, or a remote microcontroller reachable via network MQTT) rather than a serial-connected MCU:

The **`evse_board_support_API`** module provides the `evse_board_support`, `ac_rcd`, and `connector_lock` interfaces _as if_ it were a hardware driver, but instead bridges them to external MQTT topics.

**MQTT topic direction:**
```
# Commands flowing OUT to external system (EVerest→External):
everest_api/1/evse_board_support/{module_id}/e2m/enable
everest_api/1/evse_board_support/{module_id}/e2m/pwm_on
everest_api/1/evse_board_support/{module_id}/e2m/cp_state_X1
everest_api/1/evse_board_support/{module_id}/e2m/cp_state_F
everest_api/1/evse_board_support/{module_id}/e2m/allow_power_on
everest_api/1/evse_board_support/{module_id}/e2m/ac_switch_three_phases_while_charging
everest_api/1/evse_board_support/{module_id}/e2m/ac_overcurrent_limit
everest_api/1/evse_board_support/{module_id}/e2m/heartbeat

# Variables flowing IN from external system (External→EVerest):
everest_api/1/evse_board_support/{module_id}/m2e/event
everest_api/1/evse_board_support/{module_id}/m2e/capabilities
everest_api/1/evse_board_support/{module_id}/m2e/ac_nr_of_phases
everest_api/1/evse_board_support/{module_id}/m2e/ac_pp_ampacity
everest_api/1/evse_board_support/{module_id}/m2e/request_stop_transaction
everest_api/1/evse_board_support/{module_id}/m2e/rcd_current
everest_api/1/evse_board_support/{module_id}/m2e/raise_error
everest_api/1/evse_board_support/{module_id}/m2e/clear_error
everest_api/1/evse_board_support/{module_id}/m2e/communication_check
```

**Real-world usage:** The `config-CB-EVAL-DC.yaml` reference config (Texas Instruments CB-EVAL evaluation board) uses this pattern:

```yaml
cb_bsp:
  module: evse_board_support_API
  connections: {}
  config_module:
    cfg_heartbeat_interval_ms: 500
    cfg_communication_check_to_s: 5
    cfg_request_reply_to_s: 550

cb_ovm:
  module: over_voltage_monitor_API
  connections: {}
  config_module:
    cfg_heartbeat_interval_ms: 500
```

The external firmware on the CB-EVAL board connects to the same MQTT broker and handles these topics, implementing the BSP logic in its own firmware environment.

---

## 6. YAML Configuration Wiring Pattern

### Full Structure

```yaml
active_modules:
  {instance_name}:
    module: {ModuleClassName}
    config_module:
      {param}: {value}
    config_implementation:      # per-implementation config (for modules with multiple implementations)
      {impl_id}:
        {param}: {value}
    connections:
      {requirement_slot}:
        - module_id: {providing_instance}
          implementation_id: {providing_impl}
```

### Complete DC Charger Config Example

This is the annotated structure from `config-sil-dc.yaml` (simulation) and `config-CB-EVAL-DC.yaml` (real hardware), merged for clarity:

```yaml
active_modules:

  # --- Hardware BSP Driver (provides evse_board_support) ---
  yeti_driver:                        # or: cb_bsp / PhyVersoBSP / custom module
    module: YetiSimulator             # or: YetiDriver / evse_board_support_API
    config_module:
      connector_id: 1
      serial_port: /dev/ttyUSB0      # for native serial drivers
      baud_rate: 115200

  # --- Isolation Monitor (provides isolation_monitor) ---
  imd:
    module: Bender_isoCHA425HV        # or: IMDSimulator
    config_implementation:
      main:
        imd_device_id: 3
    connections:
      serial_comm_hub:
        - module_id: comm_hub
          implementation_id: main

  # --- Serial Hub (Modbus shared bus) ---
  comm_hub:
    module: SerialCommHub
    config_implementation:
      main:
        serial_port: /dev/cb_rs485
        baudrate: 19200
        parity: 2

  # --- DC Power Supply (provides power_supply_DC) ---
  powersupply_dc:
    module: InfyPower_BEG1K075G       # or: DCSupplySimulator
    config_module:
      can_device: can0                # Linux SocketCAN interface

  # --- Over-voltage Monitor (provides over_voltage_monitor) ---
  ovm:
    module: OVMSimulator              # or: hardware-specific module

  # --- SLAC (provides slac interface for ISO 15118) ---
  slac:
    module: EvseSlac                  # or: SlacSimulator
    config_implementation:
      main:
        device: cb_plc                # PLC modem network interface

  # --- ISO 15118 Stack (provides ISO15118_charger) ---
  iso15118_charger:
    module: EvseV2G
    config_module:
      device: cb_plc
      tls_security: allow
    connections:
      security:
        - module_id: evse_security
          implementation_id: main

  # --- Core EVSE Manager --- wires everything together ---
  evse_manager:
    module: EvseManager
    config_module:
      connector_id: 1
      evse_id: DE*PNX*E12345*1
      charge_mode: DC
      ac_hlc_enabled: true
      ac_hlc_use_5percent: true
      session_logging: true
    connections:
      bsp:                            # REQUIRED
        - module_id: yeti_driver
          implementation_id: board_support
      imd:                            # DC: required for CableCheck
        - module_id: imd
          implementation_id: main
      over_voltage_monitor:           # DC: required by IEC 61851-23:2023
        - module_id: ovm
          implementation_id: main
      powersupply_DC:                 # DC: required
        - module_id: powersupply_dc
          implementation_id: main
      powermeter_car_side:            # optional billing meter
        - module_id: powersupply_dc
          implementation_id: powermeter
      slac:
        - module_id: slac
          implementation_id: main
      hlc:
        - module_id: iso15118_charger
          implementation_id: charger
      # ac_rcd and connector_lock: omit for DC if handled by BSP internally
      # or wire separately:
      # ac_rcd:
      #   - module_id: yeti_driver
      #     implementation_id: rcd
      # connector_lock:
      #   - module_id: yeti_driver
      #     implementation_id: connector_lock

  # --- Auth, OCPP, Energy Management... ---
  auth:
    module: Auth
    config_module:
      selection_algorithm: FindFirst
      connection_timeout: 10
    connections:
      token_provider:
        - module_id: token_provider
          implementation_id: main
      token_validator:
        - module_id: token_validator
          implementation_id: main
      evse_manager:
        - module_id: evse_manager
          implementation_id: evse
```

---

## 7. Creating a Custom BSP Module (e.g., "EvseBoard" over CAN)

To implement a custom hardware driver for an EVSE safety controller connected via CAN:

### Step 1: Define a new module manifest

```yaml
# modules/MyEvseBoardBSP/manifest.yaml
description: BSP driver for custom EVSE board over CAN
config:
  can_device:
    description: Linux CAN interface
    type: string
    default: can0
  node_id:
    description: CAN node ID of the EVSE board
    type: integer
    default: 1
provides:
  board_support:
    interface: evse_board_support
    description: BSP interface for CP, relay, and lock control
  rcd:
    interface: ac_rcd
    description: RCD interface
  connector_lock:
    interface: connector_lock
    description: Connector lock motor
metadata:
  license: https://opensource.org/licenses/Apache-2.0
  authors:
    - Your Name
```

### Step 2: Generate skeleton with ev-cli

```bash
ev-cli module create MyEvseBoardBSP
```

### Step 3: Implement command handlers

In `board_support/evse_board_supportImpl.cpp`:

```cpp
void evse_board_supportImpl::init() {
    // Subscribe to CAN events from the board
    mod->can.on_cp_state_change([this](uint8_t state) {
        types::board_support_common::BspEvent event;
        event.event = map_cp_state(state);
        publish_event(event);
    });

    mod->can.on_relay_feedback([this](bool closed) {
        types::board_support_common::BspEvent event;
        event.event = closed ? types::board_support_common::Event::PowerOn
                             : types::board_support_common::Event::PowerOff;
        publish_event(event);
    });

    mod->can.on_caps_received([this](HwCaps c) {
        types::evse_board_support::HardwareCapabilities caps;
        caps.max_current_A_import = c.max_current;
        caps.min_current_A_import = 6.0;
        caps.max_phase_count_import = 3;
        caps.min_phase_count_import = 1;
        caps.max_current_A_export = 0;
        caps.min_current_A_export = 0;
        caps.max_phase_count_export = 1;
        caps.min_phase_count_export = 1;
        caps.supports_changing_phases_during_charging = false;
        caps.supports_cp_state_E = false;
        caps.connector_type = types::evse_board_support::Connector_type::IEC62196Type2Cable;
        publish_capabilities(caps);
    });
}

void evse_board_supportImpl::handle_pwm_on(double& value) {
    mod->can.send_set_pwm(static_cast<uint32_t>(value * 100));  // 0.01% units
}

void evse_board_supportImpl::handle_allow_power_on(
        types::evse_board_support::PowerOnOff& value) {
    mod->can.send_allow_power_on(value.allow_power_on, value.reason);
}

void evse_board_supportImpl::handle_cp_state_X1() {
    mod->can.send_set_pwm(10001);   // sentinel = X1
}

void evse_board_supportImpl::handle_cp_state_F() {
    mod->can.send_set_pwm(0);       // sentinel = F (−12V)
}

void evse_board_supportImpl::handle_enable(bool& value) {
    mod->can.send_enable(value);
}
```

### Step 4: Wire in YAML config

```yaml
active_modules:
  my_evse_board:
    module: MyEvseBoardBSP
    config_module:
      can_device: can0
      node_id: 1

  evse_manager:
    module: EvseManager
    config_module:
      charge_mode: DC
    connections:
      bsp:
        - module_id: my_evse_board
          implementation_id: board_support
      ac_rcd:
        - module_id: my_evse_board
          implementation_id: rcd
      connector_lock:
        - module_id: my_evse_board
          implementation_id: connector_lock
```

---

## 8. Safety Architecture: Dual-Control Principle

EVerest's hardware architecture explicitly acknowledges that Linux cannot guarantee real-time safety. The documented dual-control pattern:

```
┌──────────────────────────────────────────────────┐
│  Linux SBC (EVerest)                             │
│  - Session state machine                         │
│  - ISO 15118 / OCPP                              │
│  - Energy management                             │
│  - PWM duty cycle commands                       │
│  - Relay allow/deny commands                     │
└─────────────────┬────────────────────────────────┘
                  │ UART / CAN / SPI
                  ▼
┌──────────────────────────────────────────────────┐
│  Safety MCU (external controller)                │
│  - CP PWM generation (1kHz, ±12V)               │
│  - CP state detection (A/B/C/D/E/F)             │
│  - Relay driver + feedback                       │
│  - PP measurement                                │
│  - RCD monitoring                               │
│  - Overcurrent hardware trip                     │
│  - Watchdog: if Linux goes silent → open relays  │
└─────────────────┬────────────────────────────────┘
                  │ Direct electrical wiring
                  ▼
┌──────────────────────────────────────────────────┐
│  DC Contactors / AC Relays                       │
│  - Output contactors                             │
│  - IMD emergency relay (hardwired to contactor)  │
│  - OVM trip relay (hardwired to contactor)       │
└──────────────────────────────────────────────────┘
```

Key principle: **IMD and OVM emergency shutdown relays are wired electrically in series with output contactors**, bypassing the Linux stack entirely. EVerest only monitors and coordinates; the physical failsafe is independent.

---

## 9. DC Charging Sequence: Module Interaction Timeline

```
EV Plugin (CP: A→B)
    BSP: publish_event(B)
    EvseManager: enter State B
    EvseManager: bsp.lock()  [via connector_lock]
    EvseManager: bsp.pwm_on(duty_cycle)  [e.g. 5% for HLC]

SLAC (5% PWM phase)
    SLAC module handles EV↔EVSE matching
    ISO 15118: SessionSetup, ChargeParameterDiscovery

CableCheck
    EvseManager: imd.start_self_test(test_voltage_V)
    EvseManager: bsp.allow_power_on(DCCableCheck)  [relay closes at low V]
    IMD: publish self_test_result = true
    EvseManager: bsp.allow_power_on(PowerOff)  [relay opens]

PreCharge
    EvseManager: powersupply_DC.setMode(Precharge, DCPreCharge)
    EvseManager: powersupply_DC.setExportVoltageCurrent(target_V, 2A)
    EvseManager: bsp.allow_power_on(DCPreCharge)
    Monitor: voltage ramps to match battery voltage (±20V)

CurrentDemand (charging)
    EvseManager: powersupply_DC.setMode(Export, FullPowerCharging)
    EvseManager: bsp.allow_power_on(FullPowerCharging)
    EvseManager: imd.start()  [continuous monitoring]
    EnergyManager: sets enforce_limits on evse_manager
    EvseManager: powersupply_DC.setExportVoltageCurrent(V, I)  [updated per limits]

Session End
    EvseManager: powersupply_DC.setMode(Off, PowerOff)  [blocks until I<threshold]
    EvseManager: bsp.allow_power_on(PowerOff)
    EvseManager: imd.stop()
    EvseManager: bsp.pwm_on(0) or cp_state_X1()
    BSP: CP returns to B
    EvseManager: connector_lock.unlock()
    EV disconnects: CP→A
    BSP: publish_event(A)
```

---

## 10. Key GitHub Sources

| Resource | URL |
|---|---|
| Interface YAMLs | `github.com/EVerest/everest-core/tree/main/interfaces/` |
| Type definitions | `github.com/EVerest/everest-core/tree/main/types/` |
| YetiDriver (serial BSP) | `github.com/EVerest/everest-core/tree/main/modules/HardwareDrivers/EVSE/YetiDriver/` |
| PhyVersoBSP | `github.com/EVerest/everest-core/tree/main/modules/HardwareDrivers/EVSE/PhyVersoBSP/` |
| InfyPower CAN driver | `github.com/EVerest/everest-core/tree/main/modules/HardwareDrivers/PowerSupplies/InfyPower_BEG1K075G/` |
| evse_board_support_API | `github.com/EVerest/everest-core/tree/main/modules/API/evse_board_support_API/` |
| EvseManager | `github.com/EVerest/everest-core/tree/main/modules/EVSE/EvseManager/` |
| DC SIL config | `github.com/EVerest/everest-core/blob/main/config/config-sil-dc.yaml` |
| CB-EVAL-DC real config | `github.com/EVerest/everest-core/blob/main/config/config-CB-EVAL-DC.yaml` |

## Related Notes

- [[__init]] — DCFC project overview and specifications
- [[01 - Safety Philosophy]] — safety architecture principles
- [[02 - CM5 based Main Controller]] — SBC hardware for running EVerest
