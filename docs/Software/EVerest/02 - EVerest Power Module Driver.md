# EVerest Power Module Driver

Tags: #dcfc #everest #software #power-modules #can #cpp

Related: [[04 - Power Module CAN Bus Interface]] | [[01 - EVerest Safety Supervisor Integration]] | [[01 - Software Framework]] | [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]]

## 1. Overview

The `PowerModuleDriver` is a custom EVerest C++ module that runs on the Phytec SBC and bridges the EVerest `power_supply_DC` interface to the physical power modules on CAN #1. It translates high-level commands from EvseManager (set voltage, set current, change mode) into per-module CAN setpoints, while aggregating individual module status into a unified view for the rest of the EVerest stack.

This module is the single point of coordination for all paralleled 30 kW bricks: it handles current distribution, module health tracking, thermal derating compensation, and energy metering aggregation. N+1 redundancy failover is supported when `redundant_modules > 0` (e.g., larger configurations with 6+ modules); the default 150 kW configuration runs 5 active / 0 standby.

```
┌───────────────────────────────────────────────────────────────────────┐
│                          EVerest Module Graph                         │
│                                                                       │
│  ┌──────────────┐         ┌──────────────────────────────────────┐    │
│  │  EvseManager │────────▶│         PowerModuleDriver            │    │
│  │              │ power-  │                                      │    │
│  │  setMode()   │ supply  │  provides:                           │    │
│  │  setExport() │ _DC     │    main  → power_supply_DC           │    │
│  │  getCapab()  │         │    meter → powermeter                │    │
│  └──────────────┘         │                                      │    │
│                           │  internal:                           │    │
│  ┌──────────────┐         │    ModulePool  (track all bricks)    │    │
│  │    Energy    │         │    Distributor (current sharing)     │    │
│  │   Manager   │         │    CanBridge   (SocketCAN I/O)       │    │
│  │              │         │    MeterAggr   (energy totals)       │    │
│  │  enforce_   │         │                                      │    │
│  │  limits     │         └──────────────────┬───────────────────┘    │
│  └──────────────┘                           │                        │
│                                             │ SocketCAN (can1)       │
└─────────────────────────────────────────────┼────────────────────────┘
                                              │ CAN #1, 500 kbps
                                              ▼
                                   ┌────────────────────┐
                                   │  30 kW Modules ×N  │
                                   └────────────────────┘
```

## 2. EVerest Interfaces

### 2.1 power_supply_DC Interface (provided as `main`)

This is the primary interface consumed by EvseManager for DC charging control.

#### Commands (EvseManager → PowerModuleDriver)

| Command | Signature | Description |
|---------|-----------|-------------|
| `setMode` | `mode: Mode, reason: string` | Transition the power supply operating mode |
| `setExportVoltageCurrent` | `voltage: number, current: number` | Set target output voltage (V) and total current limit (A) |
| `setImportVoltageCurrent` | `voltage: number, current: number` | V2G / bidirectional (future, returns not-supported) |
| `getCapabilities` | `→ DCSupplyCapabilities` | Returns hardware min/max voltage, current, and power |

**Mode enum values:**

| Mode | Description | Module Action |
|------|-------------|---------------|
| `Off` | All modules disabled | Broadcast Mode=Off, disable all |
| `Export` | Normal power delivery | Enable active modules, apply V/I setpoints |
| `Import` | V2G bidirectional | Not supported in v1 (returns error) |
| `Fault` | Error state | Broadcast emergency disable |
| `Precharge` | Low-current bus charge | Single module in Precharge mode, rest standby |

#### Variables (PowerModuleDriver → EvseManager)

| Variable | Type | Period | Description |
|----------|------|--------|-------------|
| `voltage_current` | `VoltageCurrent` | 100 ms | Aggregate measured V and I |
| `mode` | `Mode` | On change | Current operating mode |
| `capabilities` | `DCSupplyCapabilities` | On startup | System-level hardware limits |

**`VoltageCurrent` type:**
```yaml
voltage_V: number     # Average output voltage across active modules
current_A: number     # Sum of output current from all active modules
```

**`DCSupplyCapabilities` type:**
```yaml
bidirectional: bool                    # false for v1
max_export_voltage_V: number           # e.g. 1000.0
min_export_voltage_V: number           # e.g. 200.0
max_export_current_A: number           # e.g. 375.0 (5 × 75 A)
min_export_current_A: number           # e.g. 0.0
max_export_power_W: number             # e.g. 150000 (5 × 30 kW)
current_regulation_tolerance_A: number # e.g. 2.0
peak_current_ripple_A: number          # e.g. 5.0
energy_mix_uuid: string                # optional
conversion_efficiency: number          # e.g. 0.96
```

### 2.2 powermeter Interface (provided as `meter`)

Aggregated energy metering for billing and OCPP `MeterValues`. EvseManager can connect to this as `powermeter_car_side`.

| Variable | Type | Period | Description |
|----------|------|--------|-------------|
| `powermeter` | `Powermeter` | 1000 ms | Aggregated power and energy values |

**`Powermeter` type (relevant fields):**
```yaml
timestamp: string              # ISO 8601
energy_Wh_import:
  total: number                # Cumulative session energy (Wh)
power_W:
  total: number                # Instantaneous total power (W)
voltage_V:
  DC: number                   # Output voltage
current_A:
  DC: number                   # Total output current
```

## 3. Module Manifest

```yaml
# modules/PowerModuleDriver/manifest.yaml
description: >
  CAN-based driver for paralleled 30 kW DC power modules.
  Manages current distribution, health monitoring,
  and energy metering aggregation.
config:
  can_device:
    description: Linux SocketCAN interface name
    type: string
    default: can1
  num_modules:
    description: Total number of power modules on the bus
    type: integer
    default: 5
  redundant_modules:
    description: Number of hot-standby modules (N+1)
    type: integer
    default: 0
  module_power_kw:
    description: Rated power per module
    type: number
    default: 30.0
  module_max_current_A:
    description: Maximum current per individual module
    type: number
    default: 75.0
  max_voltage_V:
    description: Maximum system output voltage
    type: number
    default: 1000.0
  min_voltage_V:
    description: Minimum system output voltage
    type: number
    default: 200.0
  max_total_current_A:
    description: Maximum total output current (all modules)
    type: number
    default: 375.0
  heartbeat_interval_ms:
    description: Master heartbeat send period
    type: integer
    default: 1000
  heartbeat_timeout_ms:
    description: Module heartbeat loss timeout
    type: integer
    default: 3000
  setpoint_cycle_ms:
    description: Setpoint broadcast period
    type: integer
    default: 100
  current_imbalance_threshold_pct:
    description: >
      Maximum allowed current deviation between modules
      before triggering rebalance (percent of per-module limit)
    type: number
    default: 15.0
  derating_headroom_pct:
    description: >
      Reserve current headroom per module to avoid hitting
      thermal limits (percent below rated max)
    type: number
    default: 5.0
provides:
  main:
    interface: power_supply_DC
    description: DC power output control for EvseManager
  meter:
    interface: powermeter
    description: Aggregated energy metering for billing
```

## 4. Source File Structure

```
modules/PowerModuleDriver/
├── manifest.yaml
├── CMakeLists.txt
├── PowerModuleDriver.hpp          # Module class: lifecycle, threads, shared state
├── PowerModuleDriver.cpp          # init(), ready(), threads
│
├── main/                          # power_supply_DC implementation
│   ├── power_supply_DCImpl.hpp
│   └── power_supply_DCImpl.cpp    # setMode, setExportVoltageCurrent, getCapabilities
│
├── meter/                         # powermeter implementation
│   ├── powermeterImpl.hpp
│   └── powermeterImpl.cpp         # Aggregated Powermeter variable publishing
│
├── lib/
│   ├── CanBridge.hpp              # SocketCAN open, TX, RX thread
│   ├── CanBridge.cpp
│   ├── ModulePool.hpp             # Per-module state tracking, health, discovery
│   ├── ModulePool.cpp
│   ├── CurrentDistributor.hpp     # Current sharing and rebalancing algorithm
│   ├── CurrentDistributor.cpp
│   ├── MeterAggregator.hpp        # Energy accumulation from per-module measurements
│   ├── MeterAggregator.cpp
│   └── CanProtocol.hpp            # CAN ID definitions, frame pack/unpack helpers
│
└── tests/
    ├── test_current_distributor.cpp
    ├── test_module_pool.cpp
    └── test_can_protocol.cpp
```

## 5. Internal Components

### 5.1 CanBridge

Thin wrapper around Linux SocketCAN. Opens the `can1` interface, spawns an RX thread, and provides a thread-safe TX method.

```cpp
// lib/CanBridge.hpp
class CanBridge {
public:
    bool open(const std::string& device);   // socket(PF_CAN, SOCK_RAW, CAN_RAW)
    void close();

    bool tx(uint32_t can_id, const uint8_t* data, uint8_t dlc);

    // RX callback: called from the RX thread for every received frame
    using RxCallback = std::function<void(uint32_t can_id,
                                          const uint8_t* data,
                                          uint8_t dlc)>;
    void set_rx_callback(RxCallback cb);

private:
    int fd_ = -1;
    std::thread rx_thread_;
    RxCallback rx_cb_;
    std::atomic<bool> running_{false};

    void rx_loop();
};
```

### 5.2 ModulePool

Tracks the state of every power module on the bus. Updated by CAN RX frames, queried by the CurrentDistributor and by EVerest interface handlers.

```cpp
// lib/ModulePool.hpp

enum class ModuleRole { Active, Standby, Offline, Faulted };

struct ModuleState {
    uint8_t node_id;
    ModuleRole role = ModuleRole::Offline;

    // From status frame (0x110 + node)
    double output_voltage_V = 0;
    double output_current_A = 0;
    uint8_t hw_state = 0;          // 0x00=Init .. 0x07=Shutdown
    uint8_t fault_flags = 0;
    uint8_t power_limit_pct = 100; // thermal derating
    uint8_t fan_speed_pct = 0;

    // From power measurement frame (0x111 + node)
    double power_kW = 0;
    double energy_kWh = 0;         // session energy

    // From thermal telemetry (0x210 + node)
    int8_t temp_mosfet = 0;
    int8_t temp_transformer = 0;
    int8_t temp_inductor = 0;
    int8_t temp_capacitor = 0;
    int8_t temp_inlet = 0;
    double efficiency = 0;

    // From fault frame (0x310 + node)
    uint8_t fault_code = 0;
    uint8_t fault_severity = 0;
    uint16_t fault_value = 0;

    // Health tracking
    std::chrono::steady_clock::time_point last_heartbeat;
    uint8_t restart_attempts = 0;

    // Assigned setpoint (written by CurrentDistributor)
    double assigned_current_A = 0;

    bool is_healthy() const;
    double available_current_A(double module_max_A, double headroom_pct) const;
};

class ModulePool {
public:
    explicit ModulePool(int num_modules, int redundant);

    void update_status(uint8_t node_id, const uint8_t* data);
    void update_power(uint8_t node_id, const uint8_t* data);
    void update_thermal(uint8_t node_id, const uint8_t* data);
    void update_heartbeat(uint8_t node_id, uint8_t nmt_state);
    void update_fault(uint8_t node_id, const uint8_t* data);

    // Queries
    std::vector<ModuleState*> active_modules();
    std::vector<ModuleState*> standby_modules();
    std::vector<ModuleState*> faulted_modules();
    int online_count() const;

    // Aggregate measurements
    double aggregate_voltage() const;    // average of active modules
    double aggregate_current() const;    // sum of active modules
    double aggregate_power_kW() const;   // sum
    double aggregate_energy_kWh() const; // sum

    // Lifecycle
    void check_heartbeats(std::chrono::milliseconds timeout);
    bool promote_standby();              // move one standby → active
    void demote_to_faulted(uint8_t node_id);

    ModuleState& operator[](uint8_t node_id);

private:
    std::map<uint8_t, ModuleState> modules_;
    int redundant_count_;
    mutable std::mutex mutex_;
};
```

### 5.3 CurrentDistributor

Implements the current sharing algorithm. Called whenever the total current demand changes, a module derates, or a module goes offline/online.

```cpp
// lib/CurrentDistributor.hpp

struct DistributionResult {
    std::map<uint8_t, double> per_module_current;  // node_id → assigned current (A)
    double achievable_current_A;                    // actual deliverable total
    bool fully_satisfied;                           // demand <= capacity
};

class CurrentDistributor {
public:
    CurrentDistributor(double module_max_A, double headroom_pct,
                       double imbalance_threshold_pct);

    // Calculate optimal distribution for given demand
    DistributionResult distribute(
        double total_demand_A,
        const std::vector<ModuleState*>& active_modules);

private:
    double module_max_A_;
    double headroom_pct_;
    double imbalance_threshold_pct_;
};
```

**Distribution algorithm:**

```
distribute(total_demand_A, active_modules):

  1. Calculate each module's available capacity:
     available[i] = module_max_A × (power_limit_pct[i] / 100) × (1 - headroom_pct)

  2. Sum total available capacity:
     total_available = Σ available[i]

  3. If total_demand ≤ total_available:
     # Proportional split: equal share, clamped to individual capacity
     base_share = total_demand / active_count
     for each module:
       assigned[i] = min(base_share, available[i])
     # Redistribute surplus from clamped modules to unclamped
     iterate until converged (max 3 iterations)

  4. If total_demand > total_available:
     # Capacity-limited: each module at its maximum available
     assigned[i] = available[i]
     achievable = total_available
     fully_satisfied = false

  5. Return per-module assignments
```

### 5.4 MeterAggregator

Accumulates energy measurements from individual module power frames into a single `Powermeter` structure for billing.

```cpp
// lib/MeterAggregator.hpp

class MeterAggregator {
public:
    void update_module(uint8_t node_id, double voltage_V, double current_A,
                       double power_kW, double energy_kWh);

    // Returns aggregated powermeter snapshot
    types::powermeter::Powermeter get_snapshot() const;

    void reset_session();  // Zero energy counters at session start

private:
    struct ModuleMeter {
        double voltage_V = 0;
        double current_A = 0;
        double power_kW = 0;
        double energy_kWh = 0;
    };

    std::map<uint8_t, ModuleMeter> module_meters_;
    double session_energy_Wh_ = 0;
    std::chrono::steady_clock::time_point session_start_;
    mutable std::mutex mutex_;
};
```

## 6. Module Lifecycle

### 6.1 init()

Called by the EVerest framework manager during startup. Opens CAN, wires CAN RX callback, and initializes internal structures.

```cpp
void PowerModuleDriver::init() {
    // Open SocketCAN interface
    if (!can_.open(config.can_device)) {
        EVLOG_error << "Failed to open CAN device: " << config.can_device;
        return;
    }

    // Initialize module pool
    pool_ = std::make_unique<ModulePool>(
        config.num_modules, config.redundant_modules);

    // Initialize current distributor
    distributor_ = std::make_unique<CurrentDistributor>(
        config.module_max_current_A,
        config.derating_headroom_pct,
        config.current_imbalance_threshold_pct);

    meter_ = std::make_unique<MeterAggregator>();

    // Wire CAN RX dispatch
    can_.set_rx_callback([this](uint32_t id, const uint8_t* data, uint8_t dlc) {
        dispatch_can_rx(id, data, dlc);
    });
}
```

### 6.2 ready()

Called after all modules in the EVerest config are initialized and connected. Starts the periodic threads: heartbeat, setpoint broadcast, and telemetry publishing.

```cpp
void PowerModuleDriver::ready() {
    // Publish capabilities to EvseManager
    p_main->publish_capabilities(build_capabilities());

    // Start background threads
    heartbeat_thread_ = std::thread(&PowerModuleDriver::heartbeat_loop, this);
    setpoint_thread_  = std::thread(&PowerModuleDriver::setpoint_loop, this);
    publish_thread_   = std::thread(&PowerModuleDriver::publish_loop, this);

    EVLOG_info << "PowerModuleDriver ready, waiting for module discovery on "
               << config.can_device;
}
```

### 6.3 Thread Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PowerModuleDriver Threads                        │
│                                                                     │
│  CAN RX Thread (spawned by CanBridge)                               │
│    Blocks on socket read → dispatch_can_rx()                        │
│    Updates: ModulePool, MeterAggregator                             │
│    Triggers: EVerest event publishing for state changes             │
│                                                                     │
│  Heartbeat Thread (1000 ms cycle)                                   │
│    Sends master heartbeat (CAN 0x700)                               │
│    Calls pool_.check_heartbeats() to detect offline modules         │
│    Promotes standby if needed                                       │
│                                                                     │
│  Setpoint Thread (100 ms cycle)                                     │
│    If mode == Export or Precharge:                                   │
│      Runs CurrentDistributor::distribute()                          │
│      Sends per-module CAN setpoint frames (0x010 + node)            │
│                                                                     │
│  Publish Thread (100 ms for V/I, 1000 ms for meter)                 │
│    Publishes voltage_current var to EvseManager                     │
│    Publishes powermeter var for billing                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

All threads access `ModulePool` and `MeterAggregator` through their internal mutexes. The setpoint thread owns the `target_voltage_` and `target_current_` atomics, which are set by the EVerest command handlers.

## 7. Command Handler Implementation

### 7.1 setMode

```cpp
void power_supply_DCImpl::handle_setMode(
        types::power_supply_DC::Mode& mode) {

    switch (mode) {
    case Mode::Off:
        mod->current_mode_ = Mode::Off;
        mod->target_current_.store(0);
        // Broadcast emergency disable
        uint8_t frame[8] = {1};  // emergency_disable = 1
        mod->can_.tx(0x0F2, frame, 8);
        // Disable all modules individually
        for (auto* m : mod->pool_->active_modules()) {
            uint8_t ctrl[8] = {0};  // enable = 0
            mod->can_.tx(0x011 + m->node_id, ctrl, 8);
        }
        publish_mode(Mode::Off);
        EVLOG_info << "Power supply mode: Off";
        break;

    case Mode::Export:
        mod->current_mode_ = Mode::Export;
        // Enable all active modules
        for (auto* m : mod->pool_->active_modules()) {
            uint8_t ctrl[8] = {1, 0, 0, 0};  // enable=1, fan=auto
            mod->can_.tx(0x011 + m->node_id, ctrl, 8);
        }
        mod->meter_->reset_session();
        publish_mode(Mode::Export);
        EVLOG_info << "Power supply mode: Export";
        break;

    case Mode::Precharge: {
        mod->current_mode_ = Mode::Precharge;
        // Enable only the first active module for precharge
        auto active = mod->pool_->active_modules();
        if (!active.empty()) {
            uint8_t ctrl[8] = {1, 0, 0, 0};
            mod->can_.tx(0x011 + active[0]->node_id, ctrl, 8);
        }
        publish_mode(Mode::Precharge);
        EVLOG_info << "Power supply mode: Precharge";
        break;
    }

    case Mode::Fault:
        mod->current_mode_ = Mode::Fault;
        mod->target_current_.store(0);
        uint8_t disable[8] = {1};
        mod->can_.tx(0x0F2, disable, 8);
        publish_mode(Mode::Fault);
        EVLOG_error << "Power supply mode: Fault";
        break;

    default:
        EVLOG_warning << "Unsupported mode requested";
        break;
    }
}
```

### 7.2 setExportVoltageCurrent

This is the core command called by EvseManager during charging to update the output setpoint. It stores the targets; the setpoint thread handles the actual CAN transmission.

```cpp
void power_supply_DCImpl::handle_setExportVoltageCurrent(
        double& voltage, double& current) {

    // Clamp to hardware limits
    voltage = std::clamp(voltage,
        mod->config.min_voltage_V, mod->config.max_voltage_V);
    current = std::clamp(current,
        0.0, mod->config.max_total_current_A);

    mod->target_voltage_.store(voltage);
    mod->target_current_.store(current);

    EVLOG_debug << "Setpoint updated: " << voltage << " V, " << current << " A";
}
```

### 7.3 getCapabilities

```cpp
types::power_supply_DC::Capabilities
power_supply_DCImpl::handle_getCapabilities() {
    int active_count = mod->pool_->online_count() - mod->config.redundant_modules;
    active_count = std::max(active_count, 0);

    return {
        .bidirectional = false,
        .max_export_voltage_V = mod->config.max_voltage_V,
        .min_export_voltage_V = mod->config.min_voltage_V,
        .max_export_current_A = active_count * mod->config.module_max_current_A,
        .min_export_current_A = 0.0,
        .max_export_power_W = active_count * mod->config.module_power_kw * 1000.0,
        .current_regulation_tolerance_A = 2.0,
        .peak_current_ripple_A = 5.0,
        .conversion_efficiency = 0.96,
    };
}
```

## 8. Setpoint Thread: CAN Transmission

The setpoint thread runs at `config.setpoint_cycle_ms` (100 ms default). It reads the current targets, runs the distribution algorithm, and broadcasts per-module setpoints.

```cpp
void PowerModuleDriver::setpoint_loop() {
    while (!should_exit_) {
        if (current_mode_ == Mode::Export || current_mode_ == Mode::Precharge) {
            auto active = pool_->active_modules();

            double voltage = target_voltage_.load();
            double current = target_current_.load();

            // In precharge, only the first module gets current
            if (current_mode_ == Mode::Precharge) {
                current = std::min(current, 2.0);  // 2A precharge limit
            }

            // Run distribution
            auto result = distributor_->distribute(current, active);

            // If capacity changed, notify EvseManager
            if (!result.fully_satisfied) {
                EVLOG_warning << "Demand " << current
                              << " A exceeds capacity " << result.achievable_current_A << " A";
            }

            // Send per-module CAN setpoints
            for (auto& [node_id, assigned_I] : result.per_module_current) {
                uint8_t frame[8] = {};
                pack_u16_le(frame, 0, static_cast<uint16_t>(voltage * 10));
                pack_u16_le(frame, 2, static_cast<uint16_t>(assigned_I * 10));

                uint8_t mode_byte = 0x02;  // CV
                if (current_mode_ == Mode::Precharge) mode_byte = 0x04;
                frame[4] = mode_byte;
                frame[5] = 0;  // default ramp rate

                can_.tx(0x010 + node_id, frame, 8);

                // Update pool with assigned value
                (*pool_)[node_id].assigned_current_A = assigned_I;
            }
        }

        std::this_thread::sleep_for(
            std::chrono::milliseconds(config.setpoint_cycle_ms));
    }
}
```

## 9. CAN RX Dispatch

```cpp
void PowerModuleDriver::dispatch_can_rx(
        uint32_t id, const uint8_t* data, uint8_t dlc) {

    // Extract message group and node ID from the CAN ID scheme
    uint8_t group   = (id >> 8) & 0x07;
    uint8_t node_id = (id >> 4) & 0x0F;
    uint8_t sub_idx = id & 0x0F;

    switch (group) {
    case 1:  // Status (0x1xx)
        if (sub_idx == 0) {
            pool_->update_status(node_id, data);
            // Check for relay feedback events
            uint8_t hw_state = data[4];
            if (hw_state == 0x06) {  // Fault
                handle_module_fault(node_id);
            }
        } else if (sub_idx == 1) {
            pool_->update_power(node_id, data);
            // Update meter aggregator
            double V = unpack_u16_le(data, 0) / 100.0;
            double I = unpack_u16_le(data, 2) / 100.0;
            double P = unpack_u16_le(data, 4) / 10.0;
            double E = unpack_u16_le(data, 6) / 100.0;
            meter_->update_module(node_id, V, I, P, E);
        }
        break;

    case 2:  // Telemetry (0x2xx)
        pool_->update_thermal(node_id, data);
        break;

    case 3:  // Fault (0x3xx)
        pool_->update_fault(node_id, data);
        handle_module_fault(node_id);
        break;

    case 7:  // Heartbeat (0x7xx), node_id is in lower bits
        pool_->update_heartbeat(id & 0x7F, data[0]);
        break;
    }
}
```

## 10. Fault Handling and Redundancy

### 10.1 Module Fault Flow

```
Module N reports fault (CAN 0x310 + N)
       │
       ▼
dispatch_can_rx() → pool_->update_fault()
       │
       ▼
handle_module_fault(node_id):
  1. pool_->demote_to_faulted(node_id)
     Module removed from active pool

  2. Is total capacity still sufficient?
     ├── Yes: redistribute current across remaining active modules
     │        setpoint_loop() picks up new distribution on next cycle
     │
     └── No: attempt promote_standby()
             ├── Standby available: promote, recalculate
             │   Log: "Module N faulted, standby Module M promoted"
             │
             └── No standby: report reduced capacity
                 publish_capabilities() with lower max current
                 EVLOG_warning << "Degraded: N active modules"

  3. Raise EVerest error if severity is Critical:
     p_main->raise_error("power_supply_DC/VendorError",
                         "Module N: fault code 0x0X")

  4. If zero modules active:
     current_mode_ = Mode::Fault
     publish_mode(Mode::Fault)
     → EvseManager terminates session
```

### 10.2 Module Recovery

When a faulted module reports healthy status again (after auto-restart or manual clear):

```
Module N heartbeat returns, status = Ready (0x01)
       │
       ▼
pool_->update_status():
  module.role transitions Faulted → Offline → (discovery)
       │
       ▼
heartbeat_loop():
  Detects new healthy module
  If active_count < (num_modules - redundant_modules):
    Assign as Active, recalculate distribution
  Else:
    Assign as Standby
  clear_error("power_supply_DC/VendorError") if no other faults
```

### 10.3 Heartbeat Loss

```
heartbeat_loop() every 1000 ms:
  pool_->check_heartbeats(config.heartbeat_timeout_ms):
    For each module:
      If now - last_heartbeat > timeout:
        module.role = Offline
        Log: "Module N heartbeat lost"
        Trigger same flow as module fault (redistribute, promote)
```

## 11. Efficiency Optimization: Module Shedding

At low power levels, running fewer modules at higher individual load is more efficient than running all modules at light load (due to fixed switching losses).

```
If current_mode_ == Export and total_demand < threshold:

  shedding_threshold = num_active * module_max_A * 0.3  (30% average load)

  If total_demand < shedding_threshold and num_active > 1:
    # Shed one module to standby
    Select module with highest temperature (thermal relief)
    Send Mode=Standby, Enable=0 to selected module
    Redistribute current across N-1 modules

  If total_demand > reactivation_threshold:
    # Bring module back from standby
    promote_standby()
    Redistribute

This optimization is configurable (disabled by default, enabled via config flag).
```

## 12. Testing

### 12.1 Unit Tests

| Test | File | Validates |
|------|------|-----------|
| Equal current split | `test_current_distributor.cpp` | N modules each get demand/N |
| Derated module redistribution | `test_current_distributor.cpp` | Surplus from clamped module distributed evenly |
| Over-demand clamping | `test_current_distributor.cpp` | `fully_satisfied = false` when demand > capacity |
| Module discovery | `test_module_pool.cpp` | Heartbeat → Ready → Active transition |
| Heartbeat timeout | `test_module_pool.cpp` | Missing heartbeat → Offline transition |
| Standby promotion | `test_module_pool.cpp` | Faulted module triggers standby → Active |
| CAN frame packing | `test_can_protocol.cpp` | Voltage/current encoding matches spec |

### 12.2 SIL Simulation

A `PowerModuleSimulator` EVerest module can simulate N power modules for software-in-the-loop testing without CAN hardware:

```yaml
# config-sil-power-modules.yaml
active_modules:
  powersupply_dc:
    module: PowerModuleSimulator
    config_module:
      num_modules: 6
      module_power_kw: 25.0
      simulate_faults: false
      simulate_derating: false
```

### 12.3 Integration Test Scenarios

| Scenario | Steps | Expected Result |
|----------|-------|-----------------|
| Normal 150 kW charge | setMode(Export), setExport(800V, 187.5A) | 5 modules each at ~37.5 A, aggregate = 187.5 A |
| Module fault mid-session | Inject fault on Module 3 during charge | Standby promoted, current redistributed, no session drop |
| Full derating | All modules at 70% thermal limit | `capabilities` updated, EvseManager reduces demand |
| Precharge | setMode(Precharge), setExport(800V, 2A) | Single module ramps bus voltage, rest in standby |
| Emergency off | setMode(Off) during full power | Broadcast 0x0F2, all modules off within 10 ms |
| Hot-swap | Remove Module 2, insert replacement | Session continues, new module discovered and added |

## 13. Related Documentation

- [[04 - Power Module CAN Bus Interface]] — CAN message dictionary, physical layer, timing analysis
- [[01 - EVerest Safety Supervisor Integration]] — How EvseManager coordinates with safety and power supply
- [[03 - Safety Supervisor Controller]] — Hardware ENABLE signal that overrides CAN commands
- [[01 - Software Framework]] — EVerest module architecture and MQTT IPC
- [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]] — EVerest `power_supply_DC` interface spec and InfyPower driver reference
- [[docs/System/01 - System Architecture|01 - System Architecture]] — 30 kW module internal architecture and control hierarchy
