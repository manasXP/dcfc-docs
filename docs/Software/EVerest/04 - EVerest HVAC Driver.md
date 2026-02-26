# EVerest HVAC Driver

Tags: #dcfc #everest #software #hvac #thermal #can #cpp

Related: [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]] | [[06 - EVerest Power Module Driver]] | [[01 - EVerest Safety Supervisor Integration]] | [[01 - Software Framework]] | [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]]

## 1. Overview

The `HvacDriver` is a custom EVerest C++ module that runs on the CM5 and bridges the HVAC clip-on unit on CAN #3 to the EVerest framework. Unlike the power module driver or safety supervisor BSP — which implement standard EVerest interfaces (`power_supply_DC`, `evse_board_support`) — the HVAC driver implements a **custom `thermal_management` interface**, since EVerest has no built-in abstraction for cabinet cooling.

The module translates high-level thermal commands (set mode, set setpoint, configure alarms) into CAN frames per the [[docs/HVAC/04 - HVAC CANBus Interface Specification|HVAC CAN protocol]], while aggregating HVAC telemetry (temperatures, pressures, fault codes) into EVerest variables consumed by EvseManager, EnergyManager, and OCPP.

Its primary role is **thermal derating coordination**: when the HVAC unit reports elevated cabinet temperatures or requests derating, the HvacDriver signals the EnergyManager to reduce charger output — protecting power electronics without requiring the safety supervisor to intervene.

```
┌───────────────────────────────────────────────────────────────────────┐
│                          EVerest Module Graph                         │
│                                                                       │
│  ┌──────────────┐         ┌──────────────────────────────────────┐    │
│  │  EvseManager │         │            HvacDriver                │    │
│  │              │         │                                      │    │
│  │  subscribes: │         │  provides:                           │    │
│  │  thermal_    │◄────────│    thermal → thermal_management      │    │
│  │  status      │         │    hvac_meter → powermeter           │    │
│  └──────────────┘         │                                      │    │
│                           │  requires:                           │    │
│  ┌──────────────┐         │    energy → energy_price_information │    │
│  │   Energy     │         │      (optional, for site budgeting)  │    │
│  │  Manager     │◄────────│                                      │    │
│  │              │         │  internal:                           │    │
│  │  hvac_power  │         │    CanBridge   (SocketCAN CAN #3)    │    │
│  │  demand      │         │    HvacState   (mode, temps, faults) │    │
│  └──────────────┘         │    DerateLogic (thermal derating)    │    │
│                           │                                      │    │
│  ┌──────────────┐         └──────────────────┬───────────────────┘    │
│  │    OCPP      │                            │                        │
│  │              │                            │ SocketCAN (can2)       │
│  │  Diagnostics │                            │ (CAN #3, 250 kbps)    │
│  │  StatusNotif │                            │                        │
│  └──────────────┘                            ▼                        │
│                                   ┌────────────────────┐              │
│                                   │  HVAC Clip-On Unit │              │
│                                   │  (STM32/RP2350)    │              │
│                                   └────────────────────┘              │
└───────────────────────────────────────────────────────────────────────┘
```

> [!note] Why a custom interface?
> EVerest's standard interfaces cover EV charging (BSP, power supply, metering) but not cabinet thermal management. The `thermal_management` interface is defined within our project's `interfaces/` directory and is specific to this DCFC design. It follows the same YAML interface contract pattern as all EVerest interfaces (commands, variables, errors).

## 2. Custom Interface: `thermal_management`

### 2.1 Interface Definition

```yaml
# interfaces/thermal_management.yaml
description: >
  Cabinet thermal management system interface. Provides temperature
  monitoring, HVAC mode control, and thermal derating signals for
  coordinating cooling with charger power output.
cmds:
  set_mode:
    description: Set HVAC operating mode
    arguments:
      mode:
        description: Requested mode
        type: string
        enum: [Off, Standby, Cooling, Heating, Defrost, Ventilation]
    result:
      type: "null"

  set_setpoint:
    description: Set cabinet temperature setpoint
    arguments:
      temperature_C:
        description: Target cabinet temperature in °C
        type: number
        minimum: 15.0
        maximum: 45.0
    result:
      type: "null"

  set_config:
    description: Set HVAC alarm thresholds and tuning parameters
    arguments:
      high_temp_alarm_C:
        description: Cabinet over-temperature alarm threshold
        type: number
      low_temp_alarm_C:
        description: Cabinet under-temperature alarm threshold
        type: number
      hysteresis_C:
        description: Compressor on/off hysteresis
        type: number
      defrost_interval_min:
        description: Forced defrost interval (0 = auto)
        type: integer
    result:
      type: "null"

  get_diagnostics:
    description: Query HVAC runtime diagnostics
    arguments: {}
    result:
      type: object
      properties:
        runtime_hours:
          type: number
        compressor_cycles:
          type: integer
        energy_consumed_kWh:
          type: number

vars:
  temperatures:
    description: All temperature readings
    type: object
    properties:
      cabinet_C:
        type: number
      condenser_C:
        type: number
      evaporator_C:
        type: number
      ambient_C:
        type: number

  status:
    description: HVAC operating status
    type: object
    properties:
      mode:
        type: string
        enum: [Off, Standby, Cooling, Heating, Defrost, Ventilation]
      compressor_speed_pct:
        type: number
      fan_internal_pct:
        type: number
      fan_external_pct:
        type: number
      power_consumption_W:
        type: number
      refrigerant_pressure_bar:
        type: number
      setpoint_reached:
        type: boolean

  derate_request:
    description: Thermal derating request from HVAC
    type: object
    properties:
      derate_requested:
        type: boolean
      derate_level:
        type: string
        enum: [None, Moderate, Severe, Shutdown]
      cabinet_temp_C:
        type: number

  fault:
    description: Active HVAC fault information
    type: object
    properties:
      fault_active:
        type: boolean
      fault_code:
        type: integer
      fault_severity:
        type: string
        enum: [None, Warning, Minor, Major, Critical]
      fault_description:
        type: string

errors:
  - HvacCommunicationFault
  - CabinetOverTemperature
  - CabinetUnderTemperature
  - CompressorFault
  - RefrigerantLeak
  - FanFailure
  - SensorFault
  - VendorError
  - VendorWarning
```

### 2.2 Custom Type: `HvacDerateLevel`

```yaml
# types/thermal_management.yaml
HvacDerateLevel:
  description: Thermal derating severity levels
  type: string
  enum:
    - None         # Normal operation, no derating
    - Moderate     # Cabinet 40-45°C, reduce output by 25%
    - Severe       # Cabinet 45-50°C, reduce output by 50%
    - Shutdown     # Cabinet >50°C, initiate orderly shutdown
```

## 3. Module Manifest

```yaml
# modules/HvacDriver/manifest.yaml
description: >
  CAN-based driver for the HVAC clip-on cooling unit.
  Manages cabinet thermal control, temperature monitoring,
  thermal derating coordination, and HVAC diagnostics.
config:
  can_device:
    description: Linux SocketCAN interface name for CAN #3
    type: string
    default: can2
  heartbeat_interval_ms:
    description: Master heartbeat send period to HVAC unit
    type: integer
    default: 2000
  heartbeat_timeout_ms:
    description: HVAC heartbeat loss timeout (3 missed cycles)
    type: integer
    default: 6000
  command_cycle_ms:
    description: HVAC command message send period
    type: integer
    default: 1000
  default_setpoint_C:
    description: Default cabinet temperature setpoint at startup
    type: number
    default: 35.0
  default_mode:
    description: Default HVAC mode at startup
    type: string
    default: Standby
  derate_moderate_temp_C:
    description: Cabinet temperature that triggers moderate derating (25%)
    type: number
    default: 40.0
  derate_severe_temp_C:
    description: Cabinet temperature that triggers severe derating (50%)
    type: number
    default: 45.0
  derate_shutdown_temp_C:
    description: Cabinet temperature that triggers orderly shutdown
    type: number
    default: 50.0
  high_temp_alarm_C:
    description: HVAC over-temperature alarm threshold
    type: number
    default: 55.0
  low_temp_alarm_C:
    description: HVAC under-temperature alarm threshold (cold climate)
    type: number
    default: -10.0
  hysteresis_C:
    description: Compressor on/off hysteresis
    type: number
    default: 2.0
  hvac_power_budget_W:
    description: Maximum HVAC power draw for site load calculation
    type: number
    default: 3000.0
provides:
  thermal:
    interface: thermal_management
    description: Cabinet thermal monitoring and HVAC control
  hvac_meter:
    interface: powermeter
    description: HVAC power consumption metering for site budgeting
```

## 4. Source File Structure

```
modules/HvacDriver/
├── manifest.yaml
├── CMakeLists.txt
├── HvacDriver.hpp                  # Module class: lifecycle, threads, shared state
├── HvacDriver.cpp                  # init(), ready(), background threads
│
├── thermal/                        # thermal_management implementation
│   ├── thermal_managementImpl.hpp
│   └── thermal_managementImpl.cpp  # set_mode, set_setpoint, get_diagnostics
│
├── hvac_meter/                     # powermeter implementation
│   ├── powermeterImpl.hpp
│   └── powermeterImpl.cpp          # HVAC power consumption publishing
│
├── lib/
│   ├── CanBridge.hpp               # SocketCAN open, TX, RX thread (shared with PowerModuleDriver)
│   ├── CanBridge.cpp
│   ├── HvacState.hpp               # HVAC telemetry state tracking
│   ├── HvacState.cpp
│   ├── DerateLogic.hpp             # Thermal derating decision engine
│   ├── DerateLogic.cpp
│   └── HvacCanProtocol.hpp         # CAN ID definitions, frame pack/unpack
│
└── tests/
    ├── test_derate_logic.cpp
    ├── test_hvac_state.cpp
    └── test_hvac_can_protocol.cpp
```

## 5. Internal Components

### 5.1 CanBridge

Same `CanBridge` class used by the PowerModuleDriver (see [[06 - EVerest Power Module Driver#5.1 CanBridge]]). Opens `can2` (CAN #3) at 250 kbps, provides thread-safe TX and callback-driven RX.

> [!note] SocketCAN device mapping
> Linux SocketCAN enumerates interfaces as `can0`, `can1`, `can2`, etc. In our CM5 configuration:
> - `can0` → CAN #2 (safety supervisor, used by SafetySupervisorBSP)
> - `can1` → CAN #1 (power modules, used by PowerModuleDriver)
> - `can2` → CAN #3 (HVAC, used by HvacDriver)

### 5.2 HvacState

Tracks all telemetry received from the HVAC unit. Updated by CAN RX frames, read by the thermal management and derating logic.

```cpp
// lib/HvacState.hpp

enum class HvacMode : uint8_t {
    Off         = 0,
    Standby     = 1,
    Cooling     = 2,
    Heating     = 3,
    Defrost     = 4,
    Ventilation = 5
};

enum class FaultSeverity : uint8_t {
    None     = 0xFF,
    Warning  = 0,
    Minor    = 1,
    Major    = 2,
    Critical = 3
};

struct HvacTemperatures {
    double cabinet_C    = 0;     // T1: from CAN 0x100, bytes 0-1
    double condenser_C  = 0;     // T2: from CAN 0x100, bytes 2-3
    double evaporator_C = 0;     // T3: from CAN 0x100, bytes 4-5
    double ambient_C    = 0;     // T4: from CAN 0x100, bytes 6-7
};

struct HvacOperating {
    HvacMode mode                = HvacMode::Off;
    uint8_t  compressor_speed    = 0;    // 0-100%
    uint8_t  fan_internal        = 0;    // 0-100%
    uint8_t  fan_external        = 0;    // 0-100%
    uint16_t power_consumption_W = 0;
    double   refrigerant_pressure_bar = 0;
    uint8_t  state_flags         = 0;    // bitfield
};

struct HvacFault {
    uint8_t  code      = 0;
    FaultSeverity severity = FaultSeverity::None;
    uint16_t value     = 0;
    uint32_t timestamp = 0;
};

struct HvacDiagnostics {
    uint32_t runtime_hours      = 0;
    uint16_t compressor_cycles  = 0;
    double   energy_consumed_kWh = 0;
};

class HvacState {
public:
    void update_temperatures(const uint8_t* data);    // CAN 0x100
    void update_operating(const uint8_t* data);       // CAN 0x101
    void update_fault(const uint8_t* data);           // CAN 0x102
    void update_diagnostics(const uint8_t* data);     // CAN 0x103
    void update_heartbeat();                           // CAN 0x7FF

    // Queries
    HvacTemperatures temperatures() const;
    HvacOperating operating() const;
    HvacFault active_fault() const;
    HvacDiagnostics diagnostics() const;
    bool is_alive(std::chrono::milliseconds timeout) const;

    // State flags helpers
    bool compressor_running() const;
    bool heater_active() const;
    bool defrost_active() const;
    bool setpoint_reached() const;
    bool derate_requested() const;
    bool overtemp_warning() const;
    bool critical_fault() const;

private:
    HvacTemperatures temps_;
    HvacOperating    oper_;
    HvacFault        fault_;
    HvacDiagnostics  diag_;
    std::chrono::steady_clock::time_point last_heartbeat_;
    mutable std::mutex mutex_;
};
```

### 5.3 DerateLogic

Evaluates cabinet temperature against configurable thresholds and produces a derating level. This is the core link between the HVAC subsystem and the charger power output.

```cpp
// lib/DerateLogic.hpp

enum class DerateLevel { None, Moderate, Severe, Shutdown };

struct DerateConfig {
    double moderate_temp_C;    // default 40°C → 25% reduction
    double severe_temp_C;      // default 45°C → 50% reduction
    double shutdown_temp_C;    // default 50°C → orderly shutdown
    double hysteresis_C;       // default 2°C → debounce transitions
};

class DerateLogic {
public:
    explicit DerateLogic(DerateConfig config);

    // Evaluate current temperature → derating decision
    DerateLevel evaluate(double cabinet_temp_C);

    // Returns the power fraction allowed (1.0 = full, 0.75, 0.5, 0.0)
    double power_fraction() const;

    // Current level
    DerateLevel current_level() const;

private:
    DerateConfig config_;
    DerateLevel  current_ = DerateLevel::None;
    double       power_fraction_ = 1.0;
};
```

**Derating algorithm:**

```
evaluate(cabinet_temp_C):

  Apply hysteresis to prevent oscillation:
    rising threshold:  nominal + 0
    falling threshold: nominal - hysteresis_C

  1. If cabinet_temp_C ≥ shutdown_temp_C:
     current_ = Shutdown
     power_fraction_ = 0.0

  2. Else if cabinet_temp_C ≥ severe_temp_C:
     current_ = Severe
     power_fraction_ = 0.5

  3. Else if cabinet_temp_C ≥ moderate_temp_C:
     current_ = Moderate
     power_fraction_ = 0.75

  4. Else if cabinet_temp_C < (moderate_temp_C - hysteresis_C):
     current_ = None
     power_fraction_ = 1.0

  (If between moderate threshold and moderate - hysteresis, hold current level)
```

```
     Cabinet Temp (°C)
         │
   ≤ 38  │    DerateLevel::None          → power_fraction = 1.00
         │    (Full power, normal cooling)
   40    │────────────────────────────── moderate threshold
         │    DerateLevel::Moderate       → power_fraction = 0.75
         │    (HVAC struggling, reduce 25%)
   45    │────────────────────────────── severe threshold
         │    DerateLevel::Severe         → power_fraction = 0.50
         │    (Thermal runaway risk, reduce 50%)
   50    │────────────────────────────── shutdown threshold
         │    DerateLevel::Shutdown       → power_fraction = 0.00
         │    (Orderly shutdown initiated)
         ▼
```

### 5.4 HvacCanProtocol

CAN frame encoding/decoding helpers matching the [[docs/HVAC/04 - HVAC CANBus Interface Specification|HVAC CAN message dictionary]].

```cpp
// lib/HvacCanProtocol.hpp

namespace HvacCan {

// CAN IDs (from HVAC CAN spec section 3.2)
constexpr uint32_t HVAC_STATUS_1     = 0x100;  // HVAC → CM5: temperatures
constexpr uint32_t HVAC_STATUS_2     = 0x101;  // HVAC → CM5: operating data
constexpr uint32_t HVAC_FAULTS       = 0x102;  // HVAC → CM5: fault report
constexpr uint32_t HVAC_DIAGNOSTICS  = 0x103;  // HVAC → CM5: runtime stats
constexpr uint32_t HVAC_COMMAND      = 0x200;  // CM5 → HVAC: mode + setpoint
constexpr uint32_t HVAC_CONFIG       = 0x201;  // CM5 → HVAC: alarm thresholds
constexpr uint32_t HEARTBEAT         = 0x7FF;  // Both: heartbeat (2000 ms)

// Temperature encoding: int16_t LE, 0.1 °C per bit
inline double decode_temp(const uint8_t* data, int offset) {
    int16_t raw = static_cast<int16_t>(data[offset] | (data[offset + 1] << 8));
    return raw / 10.0;
}

inline void encode_temp(uint8_t* data, int offset, double temp_C) {
    int16_t raw = static_cast<int16_t>(temp_C * 10);
    data[offset]     = raw & 0xFF;
    data[offset + 1] = (raw >> 8) & 0xFF;
}

// Build HVAC_Command frame (CM5 → HVAC, CAN 0x200)
inline void build_command_frame(uint8_t* frame,
                                uint8_t mode,
                                double setpoint_C,
                                uint8_t fan_override,
                                bool compressor_enable,
                                uint8_t derate_level) {
    frame[0] = mode;
    encode_temp(frame, 1, setpoint_C);
    frame[3] = fan_override;          // 0 = auto, 1-100 = manual %
    frame[4] = compressor_enable ? 1 : 0;
    frame[5] = derate_level;          // 0-3
    frame[6] = 0;                     // reserved
    frame[7] = 0;
}

// Build HVAC_Config frame (CM5 → HVAC, CAN 0x201)
inline void build_config_frame(uint8_t* frame,
                               double high_alarm_C,
                               double low_alarm_C,
                               double hysteresis_C,
                               uint8_t defrost_interval_min) {
    encode_temp(frame, 0, high_alarm_C);
    encode_temp(frame, 2, low_alarm_C);
    frame[4] = static_cast<uint8_t>(hysteresis_C / 0.5);  // 0.5°C/bit
    frame[5] = defrost_interval_min;                        // 5 min/bit, 0 = auto
    frame[6] = 0;
    frame[7] = 0;
}

// Fault code → human-readable description
inline const char* fault_description(uint8_t code) {
    switch (code) {
    case 0x00: return "No fault";
    case 0x01: return "Compressor over-current";
    case 0x02: return "Compressor locked rotor";
    case 0x03: return "High-side pressure too high";
    case 0x04: return "Low-side pressure too low";
    case 0x05: return "Cabinet over-temperature";
    case 0x06: return "Cabinet under-temperature";
    case 0x07: return "Condenser fan failure";
    case 0x08: return "Internal fan failure";
    case 0x09: return "Temperature sensor fault";
    case 0x0A: return "Communication timeout";
    case 0x0B: return "Refrigerant leak detected";
    case 0x0C: return "Heater over-temperature";
    case 0x0D: return "Power supply fault";
    default:   return "Unknown fault";
    }
}

} // namespace HvacCan
```

## 6. Module Lifecycle

### 6.1 init()

```cpp
void HvacDriver::init() {
    // Open SocketCAN interface for CAN #3
    if (!can_.open(config.can_device)) {
        EVLOG_error << "Failed to open HVAC CAN device: " << config.can_device;
        return;
    }

    // Initialize state tracker
    state_ = std::make_unique<HvacState>();

    // Initialize derating logic
    derate_ = std::make_unique<DerateLogic>(DerateConfig{
        .moderate_temp_C = config.derate_moderate_temp_C,
        .severe_temp_C   = config.derate_severe_temp_C,
        .shutdown_temp_C = config.derate_shutdown_temp_C,
        .hysteresis_C    = config.hysteresis_C
    });

    // Wire CAN RX dispatch
    can_.set_rx_callback([this](uint32_t id, const uint8_t* data, uint8_t dlc) {
        dispatch_can_rx(id, data, dlc);
    });
}
```

### 6.2 ready()

```cpp
void HvacDriver::ready() {
    // Send initial configuration to HVAC unit
    send_config();

    // Set initial mode (Standby) and setpoint
    requested_mode_.store(static_cast<uint8_t>(HvacMode::Standby));
    requested_setpoint_.store(config.default_setpoint_C);

    // Start background threads
    heartbeat_thread_ = std::thread(&HvacDriver::heartbeat_loop, this);
    command_thread_   = std::thread(&HvacDriver::command_loop, this);
    publish_thread_   = std::thread(&HvacDriver::publish_loop, this);

    EVLOG_info << "HvacDriver ready on " << config.can_device
               << ", setpoint " << config.default_setpoint_C << " °C";
}
```

### 6.3 Thread Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                       HvacDriver Threads                             │
│                                                                     │
│  CAN RX Thread (spawned by CanBridge)                               │
│    Blocks on socket read → dispatch_can_rx()                        │
│    Updates: HvacState (temps, operating, faults, diagnostics)       │
│    Triggers: fault error raising, derating evaluation               │
│                                                                     │
│  Heartbeat Thread (2000 ms cycle)                                   │
│    Sends master heartbeat (CAN 0x7FF)                               │
│    Calls state_->is_alive() to detect HVAC communication loss       │
│    Raises/clears HvacCommunicationFault error                       │
│                                                                     │
│  Command Thread (1000 ms cycle)                                     │
│    Sends HVAC_Command (CAN 0x200) with current mode + setpoint      │
│    Reads requested_mode_ and requested_setpoint_ atomics            │
│    Includes charger-side derating level in command frame             │
│                                                                     │
│  Publish Thread (1000 ms cycle)                                     │
│    Publishes temperatures var to EVerest                             │
│    Publishes status var                                              │
│    Evaluates DerateLogic and publishes derate_request var            │
│    Publishes powermeter var (HVAC power consumption)                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 7. Command Handler Implementation

### 7.1 set_mode

```cpp
void thermal_managementImpl::handle_set_mode(std::string& mode) {
    uint8_t mode_byte = 0;
    if (mode == "Off")          mode_byte = 0;
    else if (mode == "Standby") mode_byte = 1;
    else if (mode == "Cooling") mode_byte = 2;
    else if (mode == "Heating") mode_byte = 3;
    else if (mode == "Defrost") mode_byte = 4;
    else if (mode == "Ventilation") mode_byte = 5;
    else {
        EVLOG_warning << "Unknown HVAC mode: " << mode;
        return;
    }

    mod->requested_mode_.store(mode_byte);
    EVLOG_info << "HVAC mode requested: " << mode;
}
```

### 7.2 set_setpoint

```cpp
void thermal_managementImpl::handle_set_setpoint(double& temperature_C) {
    temperature_C = std::clamp(temperature_C, 15.0, 45.0);
    mod->requested_setpoint_.store(temperature_C);
    EVLOG_info << "HVAC setpoint: " << temperature_C << " °C";
}
```

### 7.3 set_config

Sends the `HVAC_Config` frame (CAN `0x201`) with updated alarm thresholds.

```cpp
void thermal_managementImpl::handle_set_config(
        double& high_temp_alarm_C, double& low_temp_alarm_C,
        double& hysteresis_C, int& defrost_interval_min) {
    uint8_t frame[8] = {};
    HvacCan::build_config_frame(frame,
        high_temp_alarm_C, low_temp_alarm_C,
        hysteresis_C,
        static_cast<uint8_t>(defrost_interval_min / 5));  // 5 min/bit
    mod->can_.tx(HvacCan::HVAC_CONFIG, frame, 8);

    EVLOG_info << "HVAC config sent: high=" << high_temp_alarm_C
               << "°C, low=" << low_temp_alarm_C << "°C";
}
```

### 7.4 get_diagnostics

```cpp
types::thermal_management::Diagnostics
thermal_managementImpl::handle_get_diagnostics() {
    auto diag = mod->state_->diagnostics();
    return {
        .runtime_hours      = static_cast<double>(diag.runtime_hours),
        .compressor_cycles  = static_cast<int>(diag.compressor_cycles),
        .energy_consumed_kWh = diag.energy_consumed_kWh
    };
}
```

## 8. CAN RX Dispatch

```cpp
void HvacDriver::dispatch_can_rx(
        uint32_t id, const uint8_t* data, uint8_t dlc) {
    switch (id) {

    case HvacCan::HVAC_STATUS_1:    // 0x100: Temperatures (1 Hz)
        state_->update_temperatures(data);
        break;

    case HvacCan::HVAC_STATUS_2:    // 0x101: Operating data (1 Hz)
        state_->update_operating(data);
        break;

    case HvacCan::HVAC_FAULTS:      // 0x102: Fault report (event-driven)
        state_->update_fault(data);
        handle_hvac_fault();
        break;

    case HvacCan::HVAC_DIAGNOSTICS: // 0x103: Runtime stats (0.2 Hz)
        state_->update_diagnostics(data);
        break;

    case HvacCan::HEARTBEAT:        // 0x7FF: Heartbeat (0.5 Hz)
        state_->update_heartbeat();
        break;
    }
}
```

## 9. Command Thread: CAN Transmission

The command thread sends the `HVAC_Command` frame at 1000 ms intervals, matching the cycle time specified in the [[docs/HVAC/04 - HVAC CANBus Interface Specification|CAN spec]].

```cpp
void HvacDriver::command_loop() {
    while (!should_exit_) {
        uint8_t mode = requested_mode_.load();
        double setpoint = requested_setpoint_.load();

        // Include charger-side derating level so HVAC knows system state
        uint8_t derate_byte = static_cast<uint8_t>(derate_->current_level());

        uint8_t frame[8] = {};
        HvacCan::build_command_frame(frame,
            mode,
            setpoint,
            0,       // fan_override = auto
            (mode == static_cast<uint8_t>(HvacMode::Cooling) ||
             mode == static_cast<uint8_t>(HvacMode::Heating)),
            derate_byte);

        can_.tx(HvacCan::HVAC_COMMAND, frame, 8);

        std::this_thread::sleep_for(
            std::chrono::milliseconds(config.command_cycle_ms));
    }
}
```

## 10. Publish Thread: Telemetry and Derating

```cpp
void HvacDriver::publish_loop() {
    while (!should_exit_) {
        auto temps = state_->temperatures();
        auto oper  = state_->operating();

        // Publish temperature readings
        p_thermal->publish_temperatures({
            .cabinet_C    = temps.cabinet_C,
            .condenser_C  = temps.condenser_C,
            .evaporator_C = temps.evaporator_C,
            .ambient_C    = temps.ambient_C
        });

        // Publish operating status
        p_thermal->publish_status({
            .mode                = mode_to_string(oper.mode),
            .compressor_speed_pct = static_cast<double>(oper.compressor_speed),
            .fan_internal_pct    = static_cast<double>(oper.fan_internal),
            .fan_external_pct    = static_cast<double>(oper.fan_external),
            .power_consumption_W = static_cast<double>(oper.power_consumption_W),
            .refrigerant_pressure_bar = oper.refrigerant_pressure_bar,
            .setpoint_reached    = state_->setpoint_reached()
        });

        // Evaluate thermal derating
        auto level = derate_->evaluate(temps.cabinet_C);
        p_thermal->publish_derate_request({
            .derate_requested = (level != DerateLevel::None),
            .derate_level     = derate_level_to_string(level),
            .cabinet_temp_C   = temps.cabinet_C
        });

        // Publish HVAC power meter (for EnergyManager site budgeting)
        p_hvac_meter->publish_powermeter({
            .timestamp = Everest::Date::to_rfc3339(std::chrono::system_clock::now()),
            .power_W   = {.total = static_cast<double>(oper.power_consumption_W)},
            .energy_Wh_import = {.total = state_->diagnostics().energy_consumed_kWh * 1000}
        });

        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
}
```

## 11. Fault Handling

### 11.1 HVAC Fault Flow

```
HVAC sends fault frame (CAN 0x102)
       │
       ▼
dispatch_can_rx() → state_->update_fault()
       │
       ▼
handle_hvac_fault():
  1. Read fault code and severity from HvacState
  2. Map to EVerest error:

     HVAC Fault Code → EVerest Error                      Action
     ──────────────────────────────────────────────────────────────
     0x01 Compressor OC     → CompressorFault             Log + OCPP
     0x02 Locked rotor      → CompressorFault             Log + OCPP
     0x03 High pressure     → CompressorFault             Log + OCPP
     0x04 Low pressure      → VendorWarning               Log only
     0x05 Cabinet over-temp → CabinetOverTemperature      Derate/shutdown
     0x06 Cabinet under-temp→ CabinetUnderTemperature     Enable heating
     0x07 Condenser fan     → FanFailure                  Derate compressor
     0x08 Internal fan      → FanFailure                  Derate charger
     0x09 Sensor fault      → SensorFault                 Use defaults
     0x0A Comm timeout      → HvacCommunicationFault      Autonomous mode
     0x0B Refrigerant leak  → RefrigerantLeak             Emergency stop
     0x0C Heater over-temp  → VendorError                 Disable heater
     0x0D Power supply      → VendorError                 HVAC shutdown

  3. Raise error on thermal interface:
     p_thermal->raise_error(mapped_error, fault_description)

  4. Publish fault variable:
     p_thermal->publish_fault({
         .fault_active = true,
         .fault_code   = code,
         .fault_severity = severity_string,
         .fault_description = HvacCan::fault_description(code)
     })

  5. If severity == Critical:
     Force derate_request to Shutdown level
     EVLOG_error << "Critical HVAC fault: " << description
```

### 11.2 Communication Loss

```
heartbeat_loop() every 2000 ms:
  1. Send master heartbeat (CAN 0x7FF, byte 0 = rolling counter)

  2. Check HVAC heartbeat:
     If !state_->is_alive(config.heartbeat_timeout_ms):
       hvac_alive = false
       raise_error("HvacCommunicationFault",
                   "HVAC unit heartbeat lost on CAN #3")
       EVLOG_error << "HVAC communication lost"
       → EvseManager receives error
       → OCPP StatusNotification("Faulted")
       → Charger derates to 50% (safety margin without thermal data)

  3. If HVAC heartbeat returns:
     clear_error("HvacCommunicationFault")
     EVLOG_info << "HVAC communication restored"
     → Resume normal operation based on actual temperatures
```

### 11.3 Fault Recovery

When the HVAC unit clears a fault (fault code returns to `0x00` in CAN `0x102`, or fault frame stops being sent):

```cpp
void HvacDriver::check_fault_cleared() {
    auto fault = state_->active_fault();
    if (fault.code == 0x00 && current_error_raised_) {
        p_thermal->clear_all_errors();
        p_thermal->publish_fault({
            .fault_active = false,
            .fault_code = 0,
            .fault_severity = "None",
            .fault_description = "No fault"
        });
        current_error_raised_ = false;
        EVLOG_info << "HVAC fault cleared";
    }
}
```

## 12. Integration with EnergyManager

The HVAC power consumption must be accounted for in the site-level power budget. The `hvac_meter` implementation publishes HVAC power draw as a `powermeter` interface, which EnergyManager uses to calculate the available power for EV charging.

```
Site Power Budget Calculation (EnergyManager):

  grid_limit          = 200 kW  (site connection)
  hvac_power          = 3 kW    (from HvacDriver.hvac_meter)
  auxiliary_power      = 1 kW    (control electronics, displays, etc.)
  available_for_charging = grid_limit - hvac_power - auxiliary_power
                         = 196 kW

  If HVAC power increases (e.g., hot day, 7 kW):
    available_for_charging = 200 - 7 - 1 = 192 kW
    EnergyManager reduces enforce_limits on EvseManager
```

Additionally, when the HvacDriver publishes a `derate_request` with level `Moderate`, `Severe`, or `Shutdown`, EvseManager (or a custom `ThermalCoordinator` module) subscribes and applies the power fraction:

```yaml
# YAML wiring for thermal coordination
active_modules:

  hvac:
    module: HvacDriver
    config_module:
      can_device: can2
      default_setpoint_C: 35.0
      derate_moderate_temp_C: 40.0
      derate_severe_temp_C: 45.0
      derate_shutdown_temp_C: 50.0

  evse_manager:
    module: EvseManager
    config_module:
      charge_mode: DC
    connections:
      # ... bsp, imd, powersupply_DC, etc.
      thermal:
        - module_id: hvac
          implementation_id: thermal

  energy_manager:
    module: EnergyManager
    connections:
      energy_trunk:
        - module_id: evse_manager
          implementation_id: energy_grid
      # HVAC power metering for site load calculation
      powermeter_external:
        - module_id: hvac
          implementation_id: hvac_meter
```

## 13. Charging Session Integration

### 13.1 Session Lifecycle Interaction

```
Session Phase       HvacDriver Action                     HVAC Unit Response
───────────────── ─────────────────────────────────────── ────────────────────────

IDLE (no session)  set_mode(Standby), setpoint = 35°C     Fans at 10%, compressor off
                   Monitor ambient temp                    Report Status every 1 s

SESSION START      set_mode(Cooling), setpoint = 30°C     Compressor starts, fans ramp
(EV plugged in)    DerateLogic: evaluate on each publish   PID loop targets 30°C cabinet

CHARGING           Monitor derate_request continuously     Full cooling active
(150 kW output)    If Moderate → signal EvseManager         Compressor 50-100% as needed
                   Publish temps + status every 1 s         HVAC power: 1-3 kW

DERATING           derate_request = Moderate               HVAC at max effort
(cabinet hot)      EvseManager reduces to 112.5 kW (75%)   May not be enough → Severe
                   If Severe → EvseManager reduces to 75 kW

SESSION END        set_mode(Standby), setpoint = 35°C     Compressor ramps down
(EV unplugged)     Reset derate to None                    Fans to minimum, monitoring

HVAC FAULT         handle_hvac_fault() → raise_error       HVAC enters safe mode
(mid-session)      EvseManager may terminate session       Autonomous cooling continues
                   OCPP: StatusNotification(Faulted)
```

### 13.2 Pre-Cooling

When EvseManager signals that a session is starting (CP state B detected), the HvacDriver can pre-emptively switch to Cooling mode before the heat load appears. This prevents the initial temperature spike when power modules start:

```cpp
// Called by EvseManager (or via subscription to session_event var)
void HvacDriver::on_session_starting() {
    requested_mode_.store(static_cast<uint8_t>(HvacMode::Cooling));
    requested_setpoint_.store(30.0);  // Aggressive pre-cool target
    EVLOG_info << "HVAC pre-cooling: session starting";
}

void HvacDriver::on_session_ended() {
    requested_mode_.store(static_cast<uint8_t>(HvacMode::Standby));
    requested_setpoint_.store(config.default_setpoint_C);
    EVLOG_info << "HVAC standby: session ended";
}
```

## 14. YAML Configuration

### 14.1 Complete Charger Config (HVAC Section)

```yaml
active_modules:

  # ── HVAC Driver (CAN #3) ──
  hvac:
    module: HvacDriver
    config_module:
      can_device: can2
      heartbeat_interval_ms: 2000
      heartbeat_timeout_ms: 6000
      command_cycle_ms: 1000
      default_setpoint_C: 35.0
      default_mode: Standby
      derate_moderate_temp_C: 40.0
      derate_severe_temp_C: 45.0
      derate_shutdown_temp_C: 50.0
      high_temp_alarm_C: 55.0
      low_temp_alarm_C: -10.0
      hysteresis_C: 2.0
      hvac_power_budget_W: 3000.0

  # ── EvseManager (connects to HVAC for thermal) ──
  evse_manager:
    module: EvseManager
    config_module:
      connector_id: 1
      charge_mode: DC
    connections:
      bsp:
        - module_id: safety_bsp
          implementation_id: board_support
      imd:
        - module_id: safety_bsp
          implementation_id: imd
      powersupply_DC:
        - module_id: powersupply_dc
          implementation_id: main
      # ... slac, hlc, etc.

  # ── Energy Manager (includes HVAC power in site budget) ──
  energy_manager:
    module: EnergyManager
    connections:
      energy_trunk:
        - module_id: evse_manager
          implementation_id: energy_grid
```

### 14.2 EvseManager HVAC Subscription

Since `thermal_management` is a custom interface not natively required by EvseManager, the thermal derating coordination is implemented via one of two approaches:

**Approach A: Custom EvseManager patch** — Add an optional `thermal` requirement to EvseManager's manifest and subscribe to `derate_request` to adjust `enforce_limits`.

**Approach B: ThermalCoordinator module** — A lightweight standalone module that subscribes to HvacDriver's `derate_request` and calls `energy_manager.set_fuse_limit()` to reduce the site allocation. This is cleaner as it avoids patching EvseManager.

```yaml
  # Approach B: ThermalCoordinator
  thermal_coordinator:
    module: ThermalCoordinator
    connections:
      thermal:
        - module_id: hvac
          implementation_id: thermal
      energy_manager:
        - module_id: energy_manager
          implementation_id: main
```

```cpp
// ThermalCoordinator.cpp — subscribes to derate_request
void ThermalCoordinator::ready() {
    r_thermal->subscribe_derate_request([this](auto derate) {
        if (derate.derate_level == "Shutdown") {
            r_energy_manager->call_set_fuse_limit(0);
            EVLOG_error << "Thermal shutdown: HVAC requests power off";
        } else if (derate.derate_level == "Severe") {
            r_energy_manager->call_set_fuse_limit(max_power_W_ * 0.5);
        } else if (derate.derate_level == "Moderate") {
            r_energy_manager->call_set_fuse_limit(max_power_W_ * 0.75);
        } else {
            r_energy_manager->call_set_fuse_limit(max_power_W_);
        }
    });
}
```

## 15. Testing

### 15.1 Unit Tests

| Test | File | Validates |
|------|------|-----------|
| Derating thresholds | `test_derate_logic.cpp` | Correct level transitions at 40/45/50°C |
| Derating hysteresis | `test_derate_logic.cpp` | No oscillation at threshold boundaries |
| Temperature decoding | `test_hvac_can_protocol.cpp` | Signed 0.1°C encoding matches CAN spec |
| Command frame build | `test_hvac_can_protocol.cpp` | Mode + setpoint encoding verified |
| Config frame build | `test_hvac_can_protocol.cpp` | Alarm threshold encoding verified |
| Heartbeat timeout | `test_hvac_state.cpp` | `is_alive()` returns false after timeout |
| Fault code mapping | `test_hvac_state.cpp` | All 13 fault codes map to correct EVerest errors |
| State flag parsing | `test_hvac_state.cpp` | Bitfield extraction for all 8 flags |

### 15.2 SIL Simulation

An `HvacSimulator` EVerest module simulates the HVAC unit for software-in-the-loop testing without CAN hardware or a physical HVAC unit:

```yaml
# config-sil-hvac.yaml
active_modules:
  hvac_sim:
    module: HvacSimulator
    config_module:
      initial_cabinet_temp_C: 25.0
      heat_rate_C_per_kW: 0.5       # °C rise per kW of charger output
      cooling_rate_C_per_s: 0.02     # °C drop per second in Cooling mode
      simulate_faults: false
      simulate_comm_loss: false

  evse_manager:
    module: EvseManager
    config_module:
      charge_mode: DC
    connections:
      # ... normal connections ...
```

The simulator models:
- Cabinet temperature rising proportional to charger output power
- Temperature falling when HVAC is in Cooling mode
- Configurable fault injection for fault path testing
- Heartbeat simulation with optional loss

### 15.3 Integration Test Scenarios

| Scenario | Steps | Expected Result |
|----------|-------|-----------------|
| Normal cooling | Start 150 kW session, ambient 35°C | HVAC switches to Cooling, cabinet holds ≤35°C |
| Moderate derating | Simulate cabinet rising to 42°C | `derate_request` = Moderate, charger reduces to 112.5 kW |
| Severe derating | Simulate cabinet rising to 47°C | `derate_request` = Severe, charger reduces to 75 kW |
| Thermal shutdown | Simulate cabinet rising to 52°C | `derate_request` = Shutdown, session terminated orderly |
| HVAC comm loss | Disconnect CAN #3 during session | `HvacCommunicationFault` raised, charger derates to 50% |
| HVAC fault mid-session | Inject compressor over-current (0x01) | `CompressorFault` raised, OCPP notified, cooling degraded |
| Cold climate startup | Ambient -15°C, charger idle | HVAC in Heating mode, cabinet maintained >5°C |
| Pre-cooling | CP state A→B (EV plugs in) | HVAC switches to Cooling before heat load appears |
| HVAC FRU swap | Remove HVAC CAN during standby | Comm loss detected, blanking plates procedure |
| Defrost during session | Evaporator <0°C for 10 min | HVAC enters Defrost, CM5 logs mode change, may pre-derate |

## 16. OCPP Integration

HVAC telemetry is exposed to the OCPP backend through standard OCPP mechanisms:

| OCPP Feature | Source | Content |
|--------------|--------|---------|
| `MeterValues` (custom measurand) | `hvac_meter` powermeter | HVAC power consumption |
| `StatusNotification` | HvacDriver errors | `Faulted` when critical HVAC fault |
| `DiagnosticsStatusNotification` | `get_diagnostics()` | Runtime hours, cycles, energy |
| `DataTransfer` (vendor-specific) | `temperatures` var | Cabinet, ambient, condenser, evaporator temps |
| `MonitoringReport` (OCPP 2.0.1) | `derate_request` var | Thermal derating level changes |

The OCPP module subscribes to HvacDriver's published variables via the normal EVerest MQTT mechanism. No direct coupling is required — the OCPP module reads variables from the module graph like any other data source.

## 17. Related Documentation

- [[docs/HVAC/04 - HVAC CANBus Interface Specification|04 - HVAC CANBus Interface Specification]] — CAN message dictionary, physical layer, control logic
- [[docs/Hardware/06 - HVAC Clip-On Unit Hardware Design|06 - HVAC Clip-On Unit Hardware Design]] — HVAC hardware design, refrigeration, mechanical interface
- [[06 - EVerest Power Module Driver]] — Parallel CAN-based EVerest driver (design pattern reference)
- [[01 - EVerest Safety Supervisor Integration]] — Safety BSP module pattern and YAML wiring
- [[03 - Safety Supervisor Controller]] — Safety state machine and thermal fault handling
- [[01 - Software Framework]] — EVerest module architecture and MQTT IPC
- [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]] — Interface contracts and driver patterns
- [[docs/System/01 - System Architecture|01 - System Architecture]] — Thermal management system architecture
