# EVerest EvseManager

Tags: #dcfc #everest #software #evse #session-management #state-machine #dc-charging

Related: [[01 - EVerest Safety Supervisor Integration]] | [[06 - EVerest Power Module Driver]] | [[07 - EVerest OCPP201 Backend Integration]] | [[08 - EVerest HVAC Driver]] | [[09 - EVerest Energy Manager]] | [[01 - Software Framework]] | [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]]

## 1. Overview

`EvseManager` (`modules/EVSE/EvseManager/`) is the central session state machine in EVerest. Every EVSE connector in the system gets its own EvseManager instance. It is the single coordination point for all subsystems involved in a charging session — BSP hardware, isolation monitoring, power supply, connector locking, energy management, protocols, and authentication — but it **does not control hardware directly**. Instead, it orchestrates subsystem modules through declared EVerest interfaces.

EvseManager owns the IEC 61851 state machine, driving CP state transitions through the BSP and coordinating the DC charging phases (CableCheck → PreCharge → CurrentDemand → Shutdown) by calling into the power supply, IMD, and contactor interfaces at the right moments. It publishes session events consumed by Auth, OCPP, and the HMI, and receives energy limits from EnergyManager that it enforces on the power supply.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           EVerest Module Graph                          │
│                                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────────────┐    │
│  │   Auth   │  │ OCPP201  │  │  Energy  │  │     EvseManager       │    │
│  │          │  │          │  │  Manager │  │                       │    │
│  │ authorize│─▶│ session_ │  │          │  │  provides:            │    │
│  │ _response│  │ event    │  │ enforce_ │  │    evse → evse_manager│    │
│  │          │  │ powermeter│ │ limits  │──▶│    energy_grid → energy│   │
│  └──────────┘  └──────────┘  └──────────┘  │                       │    │
│                                            │  requires:            │    │
│  ┌──────────────────────┐                  │    bsp (REQUIRED)     │    │
│  │  SafetySupervisorBSP │◀─────────────────│    imd (DC required)  │    │
│  │  evse_board_support  │  bsp + imd +     │    powersupply_DC     │    │
│  │  isolation_monitor   │  rcd + lock      │    connector_lock     │    │
│  │  ac_rcd              │                  │    ac_rcd             │    │
│  │  connector_lock      │                  │    hlc (ISO 15118)    │    │
│  └──────────────────────┘                  │    slac               │    │
│                                            │    over_voltage_mon.  │    │
│  ┌──────────────────────┐                  │    powermeter (×2)    │    │
│  │  PowerModuleDriver   │◀─────────────────│    store (optional)   │    │
│  │  power_supply_DC     │  powersupply_DC  │                       │    │
│  │  powermeter          │  powermeter      └───────────────────────┘    │
│  └──────────────────────┘                                               │
│                                                                         │
│  ┌──────────────────────┐                                               │
│  │ EvseV2G / ISO 15118  │◀──── hlc (ISO15118_charger)                   │
│  └──────────────────────┘                                               │
│  ┌──────────────────────┐                                               │
│  │     EvseSlac         │◀──── slac                                     │
│  └──────────────────────┘                                               │
│  ┌──────────────────────┐                                               │
│  │ OverVoltageMonitor   │◀──── over_voltage_monitor                     │
│  └──────────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────────┘
```

> [!note] Key principle
> EvseManager never self-assigns its own power limits. It publishes what the EV requests (from ISO 15118) and receives what it's allowed from EnergyManager. It never directly drives contactors — all power path commands go through the BSP, which the safety supervisor can reject. This dual separation (energy limits from above, hardware safety from below) is fundamental to the DCFC safety architecture.

## 2. Provided Interfaces

### 2.1 `evse_manager` Interface (as `evse`)

The primary interface consumed by Auth, OCPP, API, and HMI modules.

#### Commands (called by external modules on EvseManager)

| Command | Arguments | Description |
|---------|-----------|-------------|
| `get_evse` | — | Returns EVSE info (connector type, id, status) |
| `enable_disable` | `connector_id`, `cmd_source` | Enable/disable the EVSE. Sources: `OCPP`, `LocalAPI`, `FirmwareUpdate` |
| `authorize_response` | `provided_token`, `validation_result` | Auth module delivers token validation result (Accepted, Blocked, etc.) |
| `withdraw_authorization` | — | Revoke authorization (timeout, CSMS revocation, disconnect) |
| `reserve` | `reservation_id` | Mark EVSE as reserved for a specific idToken |
| `cancel_reservation` | — | Clear reservation |
| `pause_charging` | — | Request charging pause (EV remains connected, contactors stay closed) |
| `resume_charging` | — | Resume from paused state |
| `stop_transaction` | `request: StopTransactionRequest` | End session (reason: Local, Remote, DeAuthorized, EmergencyStop, etc.) |
| `force_unlock` | `connector_id` | Emergency connector release (OCPP `UnlockConnector`) |
| `external_ready_to_start_charging` | — | Signal external initialization complete (used with `waiting_for_external_ready` config) |

#### Variables (published by EvseManager)

| Variable | Type | Period | Description |
|----------|------|--------|-------------|
| `session_event` | `SessionEvent` | On change | All session state transitions — primary event stream for OCPP and Auth |
| `limits` | `Limits` | On change | Current charging limits (voltage, current, power) |
| `ev_info` | `EVInfo` | On change | EV details from ISO 15118 (EVCCID, SoC, departure time, energy request) |
| `hw_capabilities` | `HardwareCapabilities` | At startup | Republished from BSP (max/min current, phase count, connector type) |
| `powermeter` | `Powermeter` | 1000 ms | From billing-relevant meter (car-side preferred, grid-side fallback) |
| `ready` | `boolean` | At startup | EvseManager ready to accept sessions |
| `waiting_for_external_ready` | `boolean` | At startup | Signals external init delay (for phased boot) |
| `selected_protocol` | `string` | On change | Active charging protocol (IEC 61851-1, ISO 15118-2, ISO 15118-20, DIN 70121) |

#### `SessionEvent` Type

The `session_event` variable is the core event stream. Each event carries a `type` field:

| Event Type | Trigger | OCPP Mapping |
|------------|---------|--------------|
| `Enabled` | EVSE enabled via `enable_disable` | `StatusNotification(Available)` |
| `Disabled` | EVSE disabled | `StatusNotification(Unavailable)` |
| `SessionStarted` | EV connected (CP state A→B) | `StatusNotification(Preparing)` |
| `AuthRequired` | Session needs authorization | — |
| `Authorized` | Token accepted, session authorized | `TransactionEvent(Started, Authorized)` |
| `TransactionStarted` | Energy transfer begins | `TransactionEvent(Updated, ChargingStateChanged)` |
| `ChargingStarted` | Power delivery active | `TransactionEvent(Updated, ChargingStateChanged)` |
| `ChargingPausedEV` | EV requests pause (SuspendedEV) | `TransactionEvent(Updated, SuspendedEV)` |
| `ChargingPausedEVSE` | EVSE pauses (limit=0, thermal, etc.) | `TransactionEvent(Updated, SuspendedEVSE)` |
| `ChargingResumed` | Charging resumes from pause | `TransactionEvent(Updated, ChargingStateChanged)` |
| `StoppingCharging` | Shutdown sequence initiated | — |
| `TransactionFinished` | Energy transfer complete | `TransactionEvent(Ended)` |
| `SessionFinished` | EV disconnected (CP state B→A) | `StatusNotification(Available)` |
| `Error` | Fault detected | `StatusNotification(Faulted)` |
| `AllErrorsCleared` | All faults resolved | `StatusNotification(Available)` |
| `ReplugStarted` | Replug simulation (testing) | — |
| `ReplugFinished` | Replug simulation done | — |

#### Errors Raised by EvseManager

| Error | Trigger |
|-------|---------|
| `evse_manager/Internal` | Unclassified internal error |
| `evse_manager/MREC4OverCurrentFailure` | DC output overcurrent detected |
| `evse_manager/MREC5OverVoltage` | DC output overvoltage detected |
| `evse_manager/MREC9AuthorizationTimeout` | Authorization timeout expired |
| `evse_manager/PowermeterTransactionStartFailed` | Powermeter not responding at session start |
| `evse_manager/Inoperative` | EVSE disabled or in permanent fault |
| `evse_manager/MREC22ResistanceFault` | Isolation resistance below threshold during DC charging |
| `evse_manager/MREC11CableCheckFault` | CableCheck phase failed (IMD self-test or isolation) |
| `evse_manager/VoltagePlausibilityFault` | Output voltage does not match setpoint (±20V) |

### 2.2 `energy` Interface (as `energy_grid`)

Provides the energy management channel consumed by EnergyManager.

**Published variable:**

| Variable | Type | Description |
|----------|------|-------------|
| `energy_flow_request` | `EnergyFlowRequest` | What the EVSE wants — max current, max power, voltage range, based on EV demand from ISO 15118 |

**Received variable (set by EnergyManager):**

| Variable | Type | Description |
|----------|------|-------------|
| `enforce_limits` | `EnforcedLimits` | What the EVSE is allowed — time-series schedule with `total_power_W`, `ac_max_current_A` |

EvseManager clamps all power supply setpoints to the `enforce_limits` values. If `total_power_W` drops to 0, EvseManager initiates an orderly shutdown.

## 3. Required Connections

### 3.1 Module Manifest

```yaml
# modules/EVSE/EvseManager/manifest.yaml (relevant sections)
description: >
  EVSE charging session manager. Implements the IEC 61851 state machine,
  coordinates all subsystems for AC and DC charging, and publishes
  session events for OCPP and authorization.
config:
  connector_id:
    description: Connector number for this EVSE instance
    type: integer
    default: 1
  evse_id:
    description: EVSE identifier string (for OCPP reporting)
    type: string
    default: ""
  charge_mode:
    description: Charging mode
    type: string
    enum: [AC, DC]
    default: DC
  ac_hlc_enabled:
    description: Enable high-level communication (ISO 15118) on AC
    type: boolean
    default: true
  ac_hlc_use_5percent:
    description: Use 5% PWM duty cycle to signal HLC capability
    type: boolean
    default: true
  session_logging:
    description: Enable detailed session log files
    type: boolean
    default: true
  session_logging_path:
    description: Directory for session log files
    type: string
    default: /tmp/everest_session_logs
  cable_check_wait_number_of_imd_measurements:
    description: Number of consecutive good IMD readings required to pass CableCheck
    type: integer
    default: 3
  cable_check_voltage_V:
    description: Test voltage for CableCheck phase
    type: number
    default: 500.0
  precharge_current_limit_A:
    description: Maximum current during PreCharge phase
    type: number
    default: 2.0
  precharge_voltage_tolerance_V:
    description: Voltage match tolerance for PreCharge completion (±V)
    type: number
    default: 20.0
  precharge_timeout_s:
    description: PreCharge phase timeout
    type: integer
    default: 30
  voltage_plausibility_tolerance_V:
    description: Allowable voltage deviation from setpoint during charging
    type: number
    default: 20.0
  max_current_ramp_A_per_s:
    description: Maximum current ramp rate for demand changes
    type: number
    default: 50.0
  waiting_for_external_ready:
    description: Wait for external_ready_to_start_charging before accepting sessions
    type: boolean
    default: false
provides:
  evse:
    interface: evse_manager
    description: Session management interface for Auth, OCPP, and API modules
  energy_grid:
    interface: energy
    description: Energy management interface for EnergyManager
requires:
  bsp:
    interface: evse_board_support
    min_connections: 1
    max_connections: 1
  ac_rcd:
    interface: ac_rcd
    min_connections: 0
    max_connections: 1
  connector_lock:
    interface: connector_lock
    min_connections: 0
    max_connections: 1
  powermeter_grid_side:
    interface: powermeter
    min_connections: 0
    max_connections: 1
  powermeter_car_side:
    interface: powermeter
    min_connections: 0
    max_connections: 1
  slac:
    interface: slac
    min_connections: 0
    max_connections: 1
  hlc:
    interface: ISO15118_charger
    min_connections: 0
    max_connections: 1
  imd:
    interface: isolation_monitor
    min_connections: 0
    max_connections: 1
  over_voltage_monitor:
    interface: over_voltage_monitor
    min_connections: 0
    max_connections: 1
  powersupply_DC:
    interface: power_supply_DC
    min_connections: 0
    max_connections: 1
  store:
    interface: kvs
    min_connections: 0
    max_connections: 1
```

### 3.2 Connection Summary

| Requirement | Interface | Module in DCFC | Required? |
|-------------|-----------|----------------|-----------|
| `bsp` | `evse_board_support` | SafetySupervisorBSP | **Yes** (always) |
| `imd` | `isolation_monitor` | SafetySupervisorBSP | Yes (DC) |
| `powersupply_DC` | `power_supply_DC` | PowerModuleDriver | Yes (DC) |
| `over_voltage_monitor` | `over_voltage_monitor` | OverVoltageMonitor | Yes (DC, IEC 61851-23:2023) |
| `connector_lock` | `connector_lock` | SafetySupervisorBSP | Optional |
| `ac_rcd` | `ac_rcd` | SafetySupervisorBSP | Optional |
| `slac` | `slac` | EvseSlac | Yes (ISO 15118) |
| `hlc` | `ISO15118_charger` | EvseV2G | Yes (ISO 15118) |
| `powermeter_car_side` | `powermeter` | PowerModuleDriver `.meter` | Optional (preferred) |
| `powermeter_grid_side` | `powermeter` | External meter module | Optional (fallback) |
| `store` | `kvs` | PersistentStore | Optional |

## 4. DC Charging State Machine

### 4.1 State Diagram

```
                                 ┌─────────┐
                          ┌─────▶│  IDLE   │◀──────────────────────────┐
                          │      │ (CP: A) │                           │
                          │      └────┬────┘                           │
                          │           │ EV plugs in (CP: A→B)          │
                          │           ▼                                │
                          │      ┌─────────┐                           │
                          │      │PREPARING│                           │
                   EV     │      │ (CP: B) │                           │
                unplugs   │      │         │                           │
                (CP→A)    │      │ Lock connector                      │
                          │      │ Set PWM 5% (HLC)                    │
                          │      │ Wait for auth                       │
                          │      └────┬────┘                           │
                          │           │ authorize_response(Accepted)   │
                          │           ▼                                │
                          │      ┌─────────────┐                       │
                          │      │ SLAC / HLC  │                       │
                          │      │ (ISO 15118) │                       │
                          │      │             │                       │
                          │      │ SessionSetup│                       │
                          │      │ ChargeParam │                       │
                          │      │ Discovery   │                       │
                          │      └────┬────────┘                       │
                          │           │                                │
                          │           ▼                                │
                          │      ┌─────────────┐                       │
                          │      │ CABLE CHECK │                       │
                          │      │             │                       │
                          │      │ IMD self-test                       │
                          │      │ Close relay (DCCableCheck)          │
                          │      │ Wait N good readings                │
                          │      │ Open relay                          │
                          │      └────┬────────┘                       │
                          │           │ pass                           │
                          │           ▼                                │
                          │      ┌─────────────┐                       │
                          │      │ PRE-CHARGE  │                       │
                          │      │             │                       │
                          │      │ setMode(Precharge)                  │
                          │      │ setExport(target_V, 2A)             │
                          │      │ Close relay (DCPreCharge)           │
                          │      │ Ramp V to match battery ±20V        │
                          │      └────┬────────┘                       │
                          │           │ voltage matched                │
                          │           ▼                                │
                          │      ┌──────────────────┐                  │
                          │      │ CURRENT DEMAND   │                  │
                          │      │ (CHARGING)       │                  │
                          │      │                  │                  │
                          │      │ setMode(Export)  │◀─── enforce      │
                          │      │ allow_power_on   │     _limits      │
                          │      │  (FullPowerCharging)   from         │
                          │      │ imd.start()      │     Energy       │
                          │      │ setExport(V, I)  │     Manager      │
                          │      │  (clamped to     │                  │
                          │      │   enforce_limits)│                  │
                          │      └────┬───────┬────┘                   │
                          │           │       │ EV requests stop       │
                          │    pause  │       │ or fault               │
                          │           ▼       ▼                        │
                          │      ┌──────────────────┐                  │
                          │      │   SHUTDOWN       │                  │
                          │      │                  │                  │
                          │      │ setMode(Off)     │                  │
                          │      │  (blocks until   │                  │
                          │      │   I < threshold) │                  │
                          │      │ allow_power_on   │                  │
                          │      │  (PowerOff)      │                  │
                          │      │ imd.stop()       │                  │
                          │      │ Unlock connector │                  │
                          │      └────┬─────────────┘                  │
                          │           │                                │
                          └───────────┘                                │
                                                                       │
     ┌───────────────────────────────────────────────────────────┐     │
     │ FAULT (any phase)                                         │     │
     │                                                           │     │
     │ BSP error, IMD fault, OVP/OCP, comm loss, thermal         │     │
     │ → setMode(Fault) → contactors open → session_event(Error) │─────┘
     │ → OCPP: StatusNotification(Faulted)                       │
     └───────────────────────────────────────────────────────────┘
```

### 4.2 DC Charging Sequence (Module Interaction Timeline)

```
Phase           EvseManager                  CAN Bus          Safety Supervisor
─────────────── ──────────────────────────── ──────────────── ─────────────────

1. EV Plugs In                                                CP: A → B detected
                                              ◀── 0x210 ───  Publish CP state B
                State A → B
                connector_lock.lock()
                                              ─── 0x113 ──▶  Lock motor engaged
                bsp.pwm_on(5%)
                                              ─── 0x111 ──▶  Generate 5% PWM
                publish session_event(SessionStarted)

2. Authorization
                publish session_event(AuthRequired)
                ← Auth calls authorize_response(Accepted)
                publish session_event(Authorized)

3. SLAC / HLC
                SLAC module handles matching
                ISO 15118: SessionSetup
                ISO 15118: ChargeParameterDiscovery
                → receives EV battery voltage, SoC, max current

4. CableCheck
                imd.start_self_test(500V)
                                              ─── 0x121 ──▶  Begin IMD self-test
                bsp.allow_power_on(DCCableCheck)
                                              ─── 0x101 ──▶  Validate → close relays
                                                              (at low voltage)
                                              ◀── 0x210 ───  PowerOn event
                                              ◀── 0x221 ───  self_test_result = true
                Wait for N good IMD readings
                                              ◀── 0x220 ───  isolation_measurement
                CableCheck pass

5. PreCharge
                powersupply_DC.setMode(Precharge)
                powersupply_DC.setExport(target_V, 2A)
                bsp.allow_power_on(DCPreCharge)
                                              ─── 0x101 ──▶  Update reason
                Monitor voltage via power supply
                When |V_output - V_battery| < 20V:
                  PreCharge complete

6. Charging
                powersupply_DC.setMode(Export)
                bsp.allow_power_on(FullPowerCharging)
                                              ─── 0x101 ──▶  Reason = CHARGING
                imd.start()                   ─── 0x120 ──▶  Continuous IMD
                EnergyManager sets enforce_limits
                powersupply_DC.setExport(V, I)
                  (clamped to enforce_limits)
                publish session_event(ChargingStarted)
                                              ◀── 0x220 ───  Periodic isolation
                                              ◀── 0x212 ───  Periodic telemetry

7. Session End
                powersupply_DC.setMode(Off)
                  (blocks until I < 1A)
                bsp.allow_power_on(PowerOff)
                                              ─── 0x101 ──▶  SHUTDOWN sequence:
                                                              disable bricks →
                                                              open DC → open AC
                                              ◀── 0x210 ───  PowerOff event
                imd.stop()                    ─── 0x120 ──▶  Stop IMD
                publish session_event(TransactionFinished)
                connector_lock.unlock()
                                              ─── 0x113 ──▶  Unlock motor
                                                              CP: B → A (EV unplugs)
                                              ◀── 0x210 ───  CP state A
                State A (idle)
                publish session_event(SessionFinished)
```

## 5. Coordination Logic

### 5.1 BSP Event Subscription

EvseManager subscribes to the BSP's `event` variable and drives the IEC 61851 state machine:

```cpp
// In EvseManager::init()
r_bsp->subscribe_event([this](const BspEvent& event) {
    switch (event.event) {
    case Event::A:  // EV disconnected
        handle_cp_state_A();
        break;
    case Event::B:  // EV connected, not charging
        handle_cp_state_B();
        break;
    case Event::C:  // EV connected, charging (AC only)
        handle_cp_state_C();
        break;
    case Event::PowerOn:  // Relay closure confirmed
        handle_power_on();
        break;
    case Event::PowerOff: // Relay opening confirmed
        handle_power_off();
        break;
    // ... D, E, F, Disconnected
    }
});
```

### 5.2 CP PWM Control

EvseManager sets CP PWM duty cycle through the BSP:

| Duty Cycle | IEC 61851 Meaning | When Used |
|------------|-------------------|-----------|
| 0% | State F (−12V) | EVSE error, not available |
| 5% | Digital communication | HLC enabled (`ac_hlc_use_5percent: true`) |
| 10–96% | Current limit encoding | AC charging (current = duty × 0.6A) |
| Sentinel X1 | Standby (constant +12V) | Between sessions, idle |

For DC charging, the PWM is set to 5% during the HLC/ISO 15118 negotiation phase. The actual current limit is communicated digitally via ISO 15118, not via PWM encoding.

### 5.3 Contactor Control

EvseManager calls `bsp.allow_power_on(PowerOnOff)` — it **requests** contactor state, it does not command it directly. The safety supervisor validates the request and may reject it if preconditions are not met.

```yaml
PowerOnOff:
  allow_power_on: bool
  reason: DCCableCheck | DCPreCharge | FullPowerCharging | PowerOff
```

| Reason | EvseManager Intent | Safety Supervisor Action |
|--------|-------------------|------------------------|
| `DCCableCheck` | Close relay for IMD test at low V | Validate, sequence AC→precharge→DC close |
| `DCPreCharge` | Hold relay closed during V ramp | Already closed, update internal reason |
| `FullPowerCharging` | Normal charging | Update internal reason to CHARGING |
| `PowerOff` | Open relays | Sequence: disable bricks → open DC → open AC |

> [!warning] Safety boundary
> EvseManager must never directly drive contactors. A hung Linux process must not leave contactors closed. The safety supervisor independently enforces watchdog timeout — if EvseManager stops sending CAN heartbeats, the safety supervisor opens all contactors within 2 seconds.

### 5.4 Power Supply Control

EvseManager interacts with the PowerModuleDriver through the `power_supply_DC` interface:

| Call | When | Arguments |
|------|------|-----------|
| `getCapabilities()` | At startup | → Returns `DCSupplyCapabilities` (max V, I, P) |
| `setMode(Precharge)` | PreCharge phase | Single module, low current |
| `setExportVoltageCurrent(V, I)` | During charging | V from EV demand, I clamped to `enforce_limits` |
| `setMode(Export)` | CurrentDemand phase | All active modules enabled |
| `setMode(Off)` | Session end | Blocks until current < threshold |
| `setMode(Fault)` | Error detected | Emergency disable all modules |

EvseManager subscribes to `voltage_current` (100 ms updates) for monitoring and `mode` (on change) for power supply state confirmation.

### 5.5 IMD Coordination

| Phase | EvseManager Call | Safety Supervisor / IMD Response |
|-------|-----------------|----------------------------------|
| CableCheck | `imd.start_self_test(500V)` | IMD applies test voltage, publishes `self_test_result` |
| CableCheck | Wait `cable_check_wait_number_of_imd_measurements` readings | `isolation_measurement` published periodically |
| Charging | `imd.start()` | Continuous isolation monitoring begins |
| Charging | Subscribe `isolation_measurement` | If resistance < threshold → raise `MREC22ResistanceFault` |
| Session end | `imd.stop()` | Monitoring stops |

### 5.6 Energy Limit Enforcement

EvseManager publishes its `energy_flow_request` based on EV demand, then clamps its power supply setpoint to whatever EnergyManager returns:

```cpp
// Simplified setpoint calculation
void EvseManager::update_power_supply() {
    double ev_voltage = ev_target_voltage_;    // from ISO 15118
    double ev_current = ev_max_current_;       // from ISO 15118

    // Clamp to enforce_limits from EnergyManager
    double max_power = enforce_limits_.total_power_W;
    double max_current_from_power = max_power / ev_voltage;
    double clamped_current = std::min(ev_current, max_current_from_power);

    // Apply current ramp rate
    double delta = clamped_current - last_current_;
    double max_delta = max_current_ramp_A_per_s_ * dt_;
    if (std::abs(delta) > max_delta) {
        clamped_current = last_current_ + std::copysign(max_delta, delta);
    }

    r_powersupply_DC->call_setExportVoltageCurrent(ev_voltage, clamped_current);
    last_current_ = clamped_current;
}
```

### 5.7 Error Propagation

EvseManager subscribes to errors from **all** required interfaces and reacts based on severity:

| Error Severity | EvseManager Response |
|---------------|---------------------|
| `High` | Immediate `setMode(Fault)` → contactors open → `session_event(Error)` |
| `Medium` | If HLC active: maintain PWM to report fault to EV via ISO 15118, then shutdown |
| `Low` | Log warning, continue operation if possible |

Error propagation chain:

```
Hardware fault (e.g., DC overvoltage)
    │
    ▼
Safety Supervisor: F01 → opens contactors → sends CAN 0x202 (fault code)
    │
    ▼
SafetySupervisorBSP: receives CAN frame → raise_error("PermanentFault")
    │
    ▼
EvseManager: subscribes to BSP errors → enters error state
    │
    ├──▶ publish session_event(Error)
    │         │
    │         ├──▶ OCPP201: StatusNotification(Faulted) + NotifyEvent
    │         └──▶ Auth: session cleanup
    │
    └──▶ HMI: display fault indicator
```

Safety supervisor fault code mapping:

| STM32 Fault | EVerest Error | Interface |
|-------------|---------------|-----------|
| F01: DC overvoltage | `PermanentFault` | board_support |
| F02: DC overcurrent | `PermanentFault` | board_support |
| F03: IMD fault | `DeviceFault` | imd |
| F04: RCD trip | `MREC2GroundFailure` | board_support |
| F05: Contactor weld | `PermanentFault` | board_support |
| F06: Precharge timeout | `PermanentFault` | board_support |
| F08: Over-temperature | `PermanentFault` | board_support |
| F09: Door interlock | `EnclosureOpen` | board_support |
| F10: Contactor mismatch | `PermanentFault` | board_support |
| F11: CAN loss | `CommunicationFault` | board_support |
| F12: ADC failure | `PermanentFault` | board_support |

## 6. Session Lifecycle: Complete Event Flow

```
EV             EvseManager         Auth          OCPP201        EnergyMgr    PowerModule
───            ───────────         ────          ───────        ─────────    ───────────

Plugs in ────▶ SessionStarted ──▶ ─────────────▶ StatusNotification(Preparing)
               AuthRequired ────▶ request token
                                  validate
               ◀── authorize_     ───────────────────────────────────────────
               response(Accept)
               Authorized ──────────────────────▶ TransactionEvent(Started)
                                                                 ◀── energy_flow_request
               SLAC + ISO 15118                                  ──▶ enforce_limits
               CableCheck
               PreCharge                                                    setMode(Prech)
               ────────────────────────────────────────────────────────────▶ setExport(V,2A)
               Charging ─────────────────────────▶ TransactionEvent(Updated, ◀── enforce_
               Started                              ChargingStateChanged)     limits
                                                                            setMode(Export)
               ────────────────────────────────────────────────────────────▶ setExport(V,I)
               ...
               (ongoing: meter values, limit updates, SoC updates)
               ...
EV requests ──▶ StoppingCharging
stop                                                                        setMode(Off)
               TransactionFinished ─────────────▶ TransactionEvent(Ended)
               ────────────────────────────────────────────────────────────▶ (blocked until
                                                                              I<1A)
Unplugs ──────▶ SessionFinished ────────────────▶ StatusNotification(Available)
```

## 7. Interaction with HVAC / Thermal Derating

EvseManager does not connect to the HvacDriver directly. Thermal derating reaches EvseManager through the EnergyManager:

```
HvacDriver publishes derate_request (cabinet temp 42°C → Moderate)
    │
    ▼
ThermalCoordinator (or EnergyManager extension):
    Reduces site budget: 200 kW × 0.75 = 150 kW
    │
    ▼
EnergyManager: enforce_limits.total_power_W = 145 kW (minus aux)
    │
    ▼
EvseManager: clamps setExportVoltageCurrent()
    │
    ▼
PowerModuleDriver: redistributes current, may shed modules
```

Additionally, the HvacDriver pre-cools the cabinet when EvseManager signals a session starting (CP state A→B), anticipating the heat load from power module operation.

## 8. Configuration: DCFC YAML

### 8.1 Single-Connector 150 kW

```yaml
active_modules:

  # ── EVSE Manager ──
  evse_manager:
    module: EvseManager
    config_module:
      connector_id: 1
      evse_id: "DCFC-001-CCS1"
      charge_mode: DC
      ac_hlc_enabled: true
      ac_hlc_use_5percent: true
      session_logging: true
      session_logging_path: /var/log/everest/sessions
      cable_check_wait_number_of_imd_measurements: 3
      cable_check_voltage_V: 500.0
      precharge_current_limit_A: 2.0
      precharge_voltage_tolerance_V: 20.0
      precharge_timeout_s: 30
      voltage_plausibility_tolerance_V: 20.0
      max_current_ramp_A_per_s: 50.0
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
      powermeter_car_side:
        - module_id: powersupply_dc
          implementation_id: meter
      slac:
        - module_id: slac
          implementation_id: main
      hlc:
        - module_id: iso15118
          implementation_id: charger
      over_voltage_monitor:
        - module_id: ovm
          implementation_id: main

  # ── Safety BSP (CAN #2) ──
  safety_bsp:
    module: SafetySupervisorBSP
    config_module:
      can_device: can0
      node_id: 1
      heartbeat_interval_ms: 500
      heartbeat_timeout_ms: 2000

  # ── Power Modules (CAN #1) ──
  powersupply_dc:
    module: PowerModuleDriver
    config_module:
      can_device: can1
      num_modules: 6
      redundant_modules: 1
      module_power_kw: 25.0
      module_max_current_A: 62.5

  # ── Over-Voltage Monitor ──
  ovm:
    module: OverVoltageMonitor

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
      tls_security: allow
    connections:
      security:
        - module_id: evse_security
          implementation_id: main

  # ── Auth ──
  auth:
    module: Auth
    config_module:
      selection_algorithm: FindFirst
      connection_timeout: 30
    connections:
      token_provider:
        - module_id: rfid
          implementation_id: main
        - module_id: ocpp
          implementation_id: auth_provider
      token_validator:
        - module_id: ocpp
          implementation_id: auth_validator
      evse_manager:
        - module_id: evse_manager
          implementation_id: evse

  # ── OCPP 2.0.1 ──
  ocpp:
    module: OCPP201
    config_module:
      CompositeScheduleIntervalS: 30
      RequestCompositeScheduleUnit: 'W'
    connections:
      evse_manager:
        - module_id: evse_manager
          implementation_id: evse
      system:
        - module_id: system
          implementation_id: main
      security:
        - module_id: evse_security
          implementation_id: main
      auth:
        - module_id: auth
          implementation_id: main
      evse_energy_sink:
        - module_id: energy_manager
          implementation_id: main
          mapping:
            evse: 0
        - module_id: energy_manager
          implementation_id: main
          mapping:
            evse: 1

  # ── Energy Manager ──
  energy_manager:
    module: EnergyManager
    config_module:
      update_interval: 1000
    connections:
      energy_trunk:
        - module_id: evse_manager
          implementation_id: energy_grid

  # ── HVAC (CAN #3) ──
  hvac:
    module: HvacDriver
    config_module:
      can_device: can2
      default_setpoint_C: 35.0

  # ── Support modules ──
  evse_security:
    module: EvseSecurity
  system:
    module: System
  rfid:
    module: RFIDReader
```

### 8.2 Connection Map

```
                           Wiring Diagram

  EvseManager.bsp ──────────────────────▶ safety_bsp.board_support
  EvseManager.imd ──────────────────────▶ safety_bsp.imd
  EvseManager.ac_rcd ───────────────────▶ safety_bsp.rcd
  EvseManager.connector_lock ───────────▶ safety_bsp.connector_lock
  EvseManager.powersupply_DC ───────────▶ powersupply_dc.main
  EvseManager.powermeter_car_side ──────▶ powersupply_dc.meter
  EvseManager.slac ─────────────────────▶ slac.main
  EvseManager.hlc ──────────────────────▶ iso15118.charger
  EvseManager.over_voltage_monitor ─────▶ ovm.main

  Auth.token_provider[0] ──────────────▶ rfid.main
  Auth.token_provider[1] ──────────────▶ ocpp.auth_provider
  Auth.token_validator ────────────────▶ ocpp.auth_validator
  Auth.evse_manager ───────────────────▶ evse_manager.evse

  OCPP201.evse_manager ────────────────▶ evse_manager.evse
  OCPP201.evse_energy_sink[0] ─────────▶ energy_manager.main (evse=0)
  OCPP201.evse_energy_sink[1] ─────────▶ energy_manager.main (evse=1)

  EnergyManager.energy_trunk ──────────▶ evse_manager.energy_grid
```

## 9. Testing

### 9.1 SIL Configuration

```yaml
# config-sil-evse-manager.yaml
active_modules:
  evse_manager:
    module: EvseManager
    config_module:
      connector_id: 1
      charge_mode: DC
      session_logging: true
    connections:
      bsp:
        - module_id: safety_sim
          implementation_id: board_support
      imd:
        - module_id: safety_sim
          implementation_id: imd
      powersupply_DC:
        - module_id: power_sim
          implementation_id: main
      powermeter_car_side:
        - module_id: power_sim
          implementation_id: meter

  safety_sim:
    module: SafetySupervisorSimulator
    config_module:
      initial_cp_state: A
      imd_resistance_ohm: 10000000
      simulate_faults: false

  power_sim:
    module: PowerModuleSimulator
    config_module:
      num_modules: 6
      module_power_kw: 25.0

  energy_manager:
    module: EnergyManager
    connections:
      energy_trunk:
        - module_id: evse_manager
          implementation_id: energy_grid
```

### 9.2 Integration Test Scenarios

| Scenario | Steps | Expected Result |
|----------|-------|-----------------|
| Normal DC session | EV plugin → auth → charge → unplug | Full lifecycle, all events in correct order |
| CableCheck fail | IMD returns low resistance | `MREC11CableCheckFault`, session blocked |
| PreCharge timeout | Power supply fails to reach target V within 30 s | Session aborted, contactors open |
| Voltage plausibility | V_output deviates >20V from setpoint | `VoltagePlausibilityFault`, power cut |
| IMD fault mid-charge | Isolation drops during CurrentDemand | `MREC22ResistanceFault`, orderly stop |
| Safety supervisor fault | F01 (overvoltage) during charge | Contactors open, `PermanentFault`, OCPP Faulted |
| Communication loss | CAN #2 heartbeat lost | `CommunicationFault`, session terminated |
| OCPP remote stop | CSMS sends RequestStopTransaction | `stop_transaction` called, orderly shutdown |
| OCPP smart charging | SetChargingProfile limits to 75 kW | `enforce_limits` reduces, output drops |
| Thermal derating | Cabinet at 45°C, derate 50% | Output drops to 50% of demand |
| Auth timeout | No token within `connection_timeout` | `MREC9AuthorizationTimeout`, session aborted |
| Dual-connector | Both EVs charge simultaneously | Power shared proportionally via EnergyManager |
| Pause/resume | OCPP `pause_charging` then `resume_charging` | Power drops to 0, EV stays connected, resumes |
| Force unlock | OCPP `force_unlock` mid-session | Orderly stop → connector unlocked |
| Module fault + recovery | Power module fault, standby promoted | Session continues at reduced capacity |

## 10. Related Documentation

- [[01 - EVerest Safety Supervisor Integration]] — BSP module, CAN protocol mapping, contactor sequencing, error propagation
- [[06 - EVerest Power Module Driver]] — `power_supply_DC` interface, current distribution, module shedding
- [[07 - EVerest OCPP201 Backend Integration]] — OCPP session events, smart charging, authorization flow
- [[08 - EVerest HVAC Driver]] — Thermal derating signals, pre-cooling integration
- [[09 - EVerest Energy Manager]] — Energy tree, `enforce_limits`, site power budgeting
- [[03 - Safety Supervisor Controller]] — Safety state machine, fault codes, watchdog
- [[01 - Software Framework]] — EVerest microservices architecture, MQTT IPC
- [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]] — Interface specifications, BSP patterns, DC charging sequence
- [[docs/System/01 - System Architecture|01 - System Architecture]] — Control hierarchy, modular power architecture
- [[docs/Hardware/05 - DC Output Contactor and Pre-Charge Circuit|05 - DC Output Contactor and Pre-Charge Circuit]] — Physical contactor sequencing, pre-charge circuit
