# EVerest Energy Manager

Tags: #dcfc #everest #software #energy-management #smart-charging #load-balancing

Related: [[03 - EVerest OCPP201 Backend Integration]] | [[02 - EVerest Power Module Driver]] | [[04 - EVerest HVAC Driver]] | [[01 - EVerest Safety Supervisor Integration]] | [[01 - Software Framework]] | [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]]

## 1. Overview

The `EnergyManager` is a core EVerest module (`modules/EVSE/EnergyManager/`) that sits between the site's physical power constraints and the EvseManager(s). It is the single authority for how much power each EVSE is allowed to draw at any moment. It does **not** control hardware directly — it computes power budgets and publishes `enforce_limits` to EvseManager, which then instructs the PowerModuleDriver accordingly.

In a DCFC, the EnergyManager solves a continuous optimization problem: given a fixed site grid connection (e.g., 200 kW), subtract auxiliary loads (HVAC, control electronics), apply any OCPP smart charging constraints from the CSMS, account for thermal derating, and distribute the remaining power across one or more EVSEs.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EVerest Module Graph                            │
│                                                                         │
│  ┌────────────┐     ┌──────────────────────────────────────────────┐    │
│  │  OCPP201   │     │              EnergyManager                   │    │
│  │            │     │                                              │    │
│  │ evse_      │────▶│  requires:                                   │    │
│  │ energy_sink│     │    energy_trunk  → energy (from EvseManager) │    │
│  │ (composite │     │                                              │    │
│  │  schedule) │     │  provides:                                   │    │
│  └────────────┘     │    main → energy_manager                     │    │
│                     │                                              │    │
│  ┌────────────┐     │  internal:                                   │    │
│  │  Thermal   │     │    EnergyTree   (node-based power model)     │    │
│  │ Coordinator│────▶│    Optimizer    (schedule resolution)        │    │
│  │ (optional) │     │    GridNode     (site fuse / grid limit)     │    │
│  └────────────┘     │    EvseNode(s)  (per-EVSE budget)            │    │
│                     │                                              │    │
│  ┌────────────┐     └───────────────┬──────────────────────────────┘    │
│  │  HvacDriver│                     │                                   │
│  │  (hvac_    │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤  energy_trunk                     │
│  │   meter)   │  external meter     │  (enforce_limits pub/sub)         │
│  └────────────┘                     │                                   │
│                                     ▼                                   │
│                        ┌──────────────────────┐                         │
│                        │     EvseManager       │                         │
│                        │                       │                         │
│                        │  energy_grid (provides)│                        │
│                        │  → publishes energy    │                        │
│                        │    schedule requests   │                        │
│                        │                       │                         │
│                        │  receives:            │                         │
│                        │    enforce_limits      │                        │
│                        │  → clamps power       │                         │
│                        │    supply setpoint     │                        │
│                        └───────────┬────────────┘                        │
│                                    │                                    │
│                                    ▼                                    │
│                        ┌──────────────────────┐                         │
│                        │  PowerModuleDriver    │                         │
│                        │  setExportVoltageCurrent()                     │
│                        └──────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> [!note] Key principle
> EvseManager never self-assigns its own power limits. It publishes what it _wants_ (based on the EV's demand from ISO 15118) and receives what it's _allowed_ from EnergyManager. This separation ensures that site-level constraints, smart charging, and thermal derating are always enforced regardless of what the EV requests.

## 2. Energy Tree Model

EVerest models the power distribution as a **tree graph** where each node has power constraints (min/max import/export) and energy flows from root (grid) to leaves (EVSEs).

### 2.1 Tree Structure for a Single-EVSE DCFC

```
              ┌──────────────────────────────────┐
              │         GRID NODE (Root)          │
              │                                   │
              │  fuse_limit: 200 kW (site feed)   │
              │  phase_count: 3                   │
              │  voltage: 400V nominal            │
              └──────────────┬───────────────────┘
                             │
              ┌──────────────▼───────────────────┐
              │         EVSE NODE (Leaf)          │
              │                                   │
              │  EvseManager publishes:           │
              │    requested_power_W: 150000      │
              │    min_voltage_V: 200             │
              │    max_voltage_V: 1000            │
              │    max_current_A: 375             │
              │                                   │
              │  EnergyManager responds:          │
              │    enforce_limits:                │
              │      max_power_W: 150000          │
              │      max_current_A: 375           │
              └──────────────────────────────────┘
```

### 2.2 Tree Structure for a Dual-Connector DCFC

When the charger has two CCS connectors sharing a common power stack (power-sharing configuration):

```
              ┌──────────────────────────────────┐
              │         GRID NODE (Root)          │
              │                                   │
              │  fuse_limit: 200 kW               │
              └──────────────┬───────────────────┘
                             │
              ┌──────────────▼───────────────────┐
              │      CHARGER NODE (Internal)       │
              │                                   │
              │  Total power shelf: 150 kW        │
              │  (5× 30 kW modules)               │
              │  + auxiliary loads: ~5 kW          │
              └──────────┬────────┬──────────────┘
                         │        │
         ┌───────────────▼──┐  ┌──▼───────────────┐
         │   EVSE 1 (CCS1) │  │   EVSE 2 (CCS2) │
         │                  │  │                  │
         │  150 kW capable  │  │  150 kW capable  │
         │  (when alone)    │  │  (when alone)    │
         │  75 kW when both │  │  75 kW when both │
         │  charging        │  │  charging        │
         └──────────────────┘  └──────────────────┘
```

The EnergyManager dynamically allocates power between the two EVSEs based on actual demand:

| Scenario | EVSE 1 | EVSE 2 | Logic |
|----------|--------|--------|-------|
| Only EVSE 1 charging | 150 kW | 0 kW | Full allocation to active EVSE |
| Only EVSE 2 charging | 0 kW | 150 kW | Full allocation to active EVSE |
| Both charging equally | 75 kW | 75 kW | Equal split |
| EVSE 1 tapering (SoC 80%) | 30 kW | 120 kW | Surplus redirected to EVSE 2 |
| Both idle | 0 kW | 0 kW | Only auxiliary loads |

## 3. Module Manifest

```yaml
# modules/EVSE/EnergyManager/manifest.yaml
description: >
  Central energy management module. Computes power budgets from
  grid constraints, OCPP smart charging profiles, and thermal limits,
  then enforces limits on connected EvseManagers.
config:
  nominal_ac_voltage:
    description: Nominal AC line-to-line voltage for power calculation (3-wire system, no neutral)
    type: number
    default: 400.0
  update_interval:
    description: Period (ms) to re-evaluate and publish energy limits
    type: integer
    default: 1000
  schedule_interval_duration:
    description: Length of each schedule interval in seconds
    type: integer
    default: 60
  schedule_total_duration:
    description: Total look-ahead for composite schedule in seconds
    type: integer
    default: 600
  slice_ampere:
    description: Ampere resolution for schedule optimization
    type: number
    default: 0.5
  slice_watt:
    description: Watt resolution for schedule optimization
    type: number
    default: 500.0
  debug:
    description: Enable detailed energy manager logging
    type: boolean
    default: false
provides:
  main:
    interface: energy_manager
    description: Energy management control API
requires:
  energy_trunk:
    interface: energy
    min_connections: 1
    max_connections: 128
    description: >
      One connection per EvseManager. Each EvseManager provides
      the 'energy_grid' implementation of the 'energy' interface.
```

## 4. EVerest Interfaces

### 4.1 `energy` Interface (EvseManager → EnergyManager)

Each EvseManager provides an `energy_grid` implementation of the `energy` interface. This is the bidirectional channel between EvseManager and EnergyManager.

**Variables published by EvseManager (energy requests):**

| Variable | Type | Description |
|----------|------|-------------|
| `energy_flow_request` | `EnergyFlowRequest` | Tree structure describing what the EVSE wants |

**The `EnergyFlowRequest` contains:**

```yaml
EnergyFlowRequest:
  uuid: string                     # Unique node ID
  children: [EnergyFlowRequest]    # Sub-nodes (for composite tree)
  node_type: Evse | Generic
  priority_request:                # Immediate need
    import:
      max_current_A: number        # From EV demand (ISO 15118)
      max_phase_count: integer
      min_current_A: number
    export:                        # V2G (future)
      max_current_A: number
  optimizer_target:                # Schedule-based target
    import:
      max_current_A: number
      max_power_W: number
    export:
      max_current_A: number
      max_power_W: number
  schedule_import: [ScheduleReqEntry]  # Time-series request
  schedule_export: [ScheduleReqEntry]
```

**Variables set by EnergyManager (enforced limits):**

| Variable | Type | Description |
|----------|------|-------------|
| `enforce_limits` | `EnforcedLimits` | The power budget the EVSE must respect |

```yaml
EnforcedLimits:
  uuid: string                     # Must match the node's uuid
  valid_until: string              # ISO 8601 expiry
  schedule:                        # Time-series limit
    - timestamp: string
      limits_to_root:
        ac_max_current_A: number
        ac_max_phase_count: integer
        total_power_W: number
      limits_to_leaves:
        ac_max_current_A: number
        total_power_W: number
  schedule_import: [ScheduleResEntry]
  schedule_export: [ScheduleResEntry]
```

### 4.2 `energy_manager` Interface (provided by EnergyManager)

Allows external modules to query and control energy management:

| Command | Description |
|---------|-------------|
| `get_energy_tree` | Returns the current energy tree state |

### 4.3 `external_energy_limits` Interface (OCPP → EnergyManager)

The OCPP201 module publishes composite smart charging schedules to this interface:

| Command | Arguments | Description |
|---------|-----------|-------------|
| `set_external_limits` | `energy_flow_request: EnergyFlowRequest` | Apply OCPP composite schedule limits |

The OCPP module creates one connection for `evseId=0` (station-level) and one per EVSE (`evseId=1`, `evseId=2`, etc.). The EnergyManager merges these external limits with the tree's own constraints.

## 5. Optimization Algorithm

### 5.1 Algorithm Overview

Every `update_interval` (default 1000 ms), the EnergyManager:

```
1. COLLECT
   - Read energy_flow_request from each EvseManager
   - Read external_energy_limits from OCPP (if connected)
   - Read site fuse limit (configured or dynamic)

2. BUILD TREE
   - Construct energy tree: GridNode → [EvseNode_1, EvseNode_2, ...]
   - Annotate each node with its constraints

3. RESOLVE LIMITS
   - Top-down: propagate site limit through tree
   - Apply OCPP composite schedule (most restrictive wins)
   - Account for auxiliary power (HVAC, controls)
   - Apply thermal derating fraction (if active)

4. OPTIMIZE DISTRIBUTION
   - For each EVSE leaf, compute enforce_limits:
     max_power_W = min(
       EVSE request,
       available from parent node,
       OCPP schedule limit,
       thermal derate limit
     )

5. PUBLISH
   - Set enforce_limits on each EvseManager
   - EvseManagers clamp their power supply setpoints
```

### 5.2 Constraint Stacking (Most Restrictive Wins)

Multiple sources can constrain the power output. The EnergyManager applies all constraints and the most restrictive wins:

```
Power Output = min(
  EV_demand,             ← from ISO 15118 ChargeParameterDiscovery
  hardware_capability,   ← from PowerModuleDriver.getCapabilities()
  site_fuse_limit - aux, ← grid connection minus HVAC and controls
  OCPP_schedule_limit,   ← from CSMS composite schedule
  thermal_derate_limit   ← from ThermalCoordinator/HvacDriver
)
```

Example scenario:

```
  EV requests:          150 kW (ISO 15118)
  Hardware capability:  150 kW (5× 30 kW modules)
  Site fuse:            200 kW
  Auxiliary load:       5 kW (HVAC 3 kW + controls 2 kW)
  OCPP schedule:        100 kW (demand response event)
  Thermal derating:     None (cabinet at 33°C)

  Available = min(150, 150, 200-5, 100, ∞) = 100 kW

  Later: OCPP schedule expires:
  Available = min(150, 150, 195, ∞, ∞) = 150 kW

  Later: cabinet reaches 42°C, thermal derate to 75%:
  Available = min(150, 150, 195, ∞, 112.5) = 112.5 kW
```

### 5.3 Schedule Resolution

The EnergyManager works with time-series schedules, not just instantaneous values. Both OCPP and the EV (via ISO 15118) can provide charging schedules with periods:

```
  Time (min)    OCPP Limit    EV Request    HVAC Load    Enforce
  ───────────── ───────────── ───────────── ──────────── ──────────
  0–15          150 kW        150 kW        3 kW         147 kW
  15–30         150 kW        120 kW        4 kW         120 kW
  30–60         100 kW        100 kW        5 kW         95 kW
  60–90         80 kW         60 kW         3 kW         60 kW
  90+           150 kW        30 kW         2 kW         30 kW
```

The `schedule_interval_duration` (60 s) and `schedule_total_duration` (600 s) config params control how far ahead and at what resolution the schedule is computed.

## 6. Site Power Budgeting

### 6.1 Auxiliary Load Accounting

The HVAC power consumption is a significant variable load that must be subtracted from the site budget. The HvacDriver publishes its power draw via the `hvac_meter` powermeter interface:

```
Site Budget Calculation (every update_interval):

  grid_capacity        = site_fuse_limit_kW       (e.g., 200 kW)
  hvac_power           = hvac_meter.power_W.total  (e.g., 3-7 kW)
  control_power        = config.auxiliary_power_W   (e.g., 1-2 kW)

  available_for_EVSEs  = grid_capacity - hvac_power - control_power

  If 1 EVSE active:
    EVSE_1.enforce_limits.total_power_W = available_for_EVSEs

  If 2 EVSEs active:
    Split available_for_EVSEs based on demand ratio
    (see Section 6.2)
```

### 6.2 Dynamic Power Sharing (Dual-Connector)

When two EVSEs share a common power stack, the EnergyManager allocates power proportionally to demand, with a minimum allocation floor:

```
Dynamic Allocation Algorithm:

  P_available = grid_capacity - aux_loads
  P_hw_max    = total_module_power  (e.g., 150 kW)
  P_available = min(P_available, P_hw_max)

  demand_1 = EVSE_1 energy_flow_request.max_power_W
  demand_2 = EVSE_2 energy_flow_request.max_power_W
  demand_total = demand_1 + demand_2

  If demand_total ≤ P_available:
    # Both can be fully satisfied
    alloc_1 = demand_1
    alloc_2 = demand_2
  Else:
    # Proportional allocation
    ratio_1 = demand_1 / demand_total
    ratio_2 = demand_2 / demand_total
    alloc_1 = P_available × ratio_1
    alloc_2 = P_available × ratio_2

  # Enforce minimum allocation (prevent starvation)
  min_alloc = 10 kW  (enough to maintain ISO 15118 session)
  alloc_1 = max(alloc_1, min_alloc) if EVSE_1 is charging
  alloc_2 = max(alloc_2, min_alloc) if EVSE_2 is charging

  EVSE_1.enforce_limits.total_power_W = alloc_1
  EVSE_2.enforce_limits.total_power_W = alloc_2
```

Typical power-sharing timeline:

```
Time    Event                 EVSE 1      EVSE 2      Total
──────  ────────────────────  ──────────  ──────────  ──────
0:00    EV1 plugs in          150 kW      -           150 kW
0:08    EV2 plugs in          75 kW       75 kW       150 kW
0:15    EV1 at SoC 50%        100 kW      50 kW → 75  150 kW
        (demand drops)        (surplus    redistributed)
0:25    EV1 at SoC 80%        30 kW       120 kW      150 kW
        (tapers heavily)
0:35    EV1 unplugs            -          150 kW      150 kW
0:50    EV2 at SoC 80%        -           50 kW       50 kW
```

## 7. Thermal Derating Integration

The EnergyManager receives thermal derating signals either through:

**Approach A** — Direct subscription to HvacDriver's `derate_request` variable (requires custom EnergyManager extension)

**Approach B** — A standalone `ThermalCoordinator` module that subscribes to `derate_request` and calls the EnergyManager's fuse limit API (documented in [[04 - EVerest HVAC Driver#14.2 EvseManager HVAC Subscription]])

Either way, the effect is the same: the available power budget is reduced by the derating fraction:

```
Thermal Derating Applied to Energy Tree:

  DerateLevel::None      → power_fraction = 1.00 → no effect
  DerateLevel::Moderate  → power_fraction = 0.75 → reduce 25%
  DerateLevel::Severe    → power_fraction = 0.50 → reduce 50%
  DerateLevel::Shutdown  → power_fraction = 0.00 → orderly stop

  effective_limit = site_fuse_limit × power_fraction - aux_loads

  Example (Moderate derating, 200 kW site):
    effective_limit = 200 × 0.75 - 5 = 145 kW
    (vs. normal: 200 - 5 = 195 kW)
```

The EnergyManager publishes the reduced `enforce_limits` to EvseManager, which clamps the PowerModuleDriver setpoint. This happens entirely within the EVerest module graph — no safety supervisor involvement.

If thermal conditions worsen to `Shutdown`, the EnergyManager sets `enforce_limits.total_power_W = 0`, causing EvseManager to initiate an orderly session termination (ramp down current, open contactors via safety supervisor).

## 8. OCPP Smart Charging Integration

The OCPP201 module calculates composite charging schedules from all active `ChargingProfile`s and publishes them to EnergyManager via the `external_energy_limits` interface. See [[03 - EVerest OCPP201 Backend Integration#6. Smart Charging]] for the full OCPP profile structure.

### 8.1 Data Flow

```
CSMS sends SetChargingProfile.req (e.g., 100 kW for 1 hour)
    │
    ▼
OCPP201 module: libocpp stores profile in SQLite
    │
    ▼ (every CompositeScheduleIntervalS = 30 s)
libocpp calculates composite schedule
    │
    ▼
OCPP201 calls set_external_limits on evse_energy_sink
    │
    ▼
EnergyManager receives external schedule
    │
    ▼ (every update_interval = 1000 ms)
EnergyManager merges external limits with tree constraints
    │
    ▼
EnergyManager publishes enforce_limits to EvseManager
    │
    ▼
EvseManager clamps power supply setpoint
    │
    ▼
PowerModuleDriver adjusts per-module current distribution
    │ (may shed modules to standby for efficiency)
    ▼
Actual output matches OCPP schedule limit
```

### 8.2 Profile Priority (Stack Levels)

When multiple profiles are active, libocpp resolves them before passing to EnergyManager:

| Purpose | Applied To | Priority |
|---------|-----------|----------|
| `ChargingStationMaxProfile` | evseId = 0 | Highest (station-wide cap) |
| `ChargingStationExternalConstraints` | evseId = 0 | Grid operator constraint |
| `TxDefaultProfile` | Per EVSE or station | Default when no TxProfile |
| `TxProfile` | Active transaction | Per-session limit |

The most restrictive limit at each time point wins. The composite schedule — already the minimum of all applicable profiles — arrives at EnergyManager as a single time-series.

### 8.3 GetCompositeSchedule

The CSMS can query the effective schedule via `GetCompositeSchedule.req`. This is useful for verifying that limits are applied correctly. EVerest/libocpp responds with the computed composite schedule, reflecting all profiles and local constraints.

## 9. Module Shedding Coordination

The EnergyManager's `enforce_limits` directly affects how many power modules need to be active. The PowerModuleDriver implements module shedding (documented in [[02 - EVerest Power Module Driver#11. Efficiency Optimization: Module Shedding]]) based on the actual demand:

```
enforce_limits = 75 kW
    │
    ▼
EvseManager: setExportVoltageCurrent(800V, 93.75A)
    │
    ▼
PowerModuleDriver:
  total_demand = 93.75A
  per_module_max = 75.0A
  modules_needed = ceil(93.75 / 75.0) = 2
  active = 2, standby = 3

  → 2 modules active at 46.88A each (63% load → near peak efficiency ~97%)
  → 3 modules in standby (fans off, no switching losses)
  → Saves ~1.2 kW in losses vs. running all 5 at 25% load
```

## 10. YAML Configuration

### 10.1 Single-EVSE DCFC (150 kW)

```yaml
active_modules:

  # ── Energy Manager ──
  energy_manager:
    module: EnergyManager
    config_module:
      nominal_ac_voltage: 230.0
      update_interval: 1000
      schedule_interval_duration: 60
      schedule_total_duration: 600
      slice_watt: 500.0
      debug: false
    connections:
      energy_trunk:
        - module_id: evse_manager
          implementation_id: energy_grid

  # ── EVSE Manager (requires energy management from tree) ──
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
      powermeter_car_side:
        - module_id: powersupply_dc
          implementation_id: meter
      # ... slac, hlc, connector_lock, ovm ...

  # ── OCPP Smart Charging (publishes schedules to EnergyManager) ──
  ocpp:
    module: OCPP201
    config_module:
      CompositeScheduleIntervalS: 30
      RequestCompositeScheduleDurationS: 600
      RequestCompositeScheduleUnit: 'W'
    connections:
      evse_manager:
        - module_id: evse_manager
          implementation_id: evse
      evse_energy_sink:
        - module_id: energy_manager
          implementation_id: main
          mapping:
            evse: 0         # Station-level profiles
        - module_id: energy_manager
          implementation_id: main
          mapping:
            evse: 1         # EVSE 1 profiles
      # ... system, security, auth ...
```

### 10.2 Dual-Connector DCFC (150 kW Shared)

```yaml
active_modules:

  energy_manager:
    module: EnergyManager
    config_module:
      nominal_ac_voltage: 230.0
      update_interval: 1000
    connections:
      energy_trunk:
        - module_id: evse_manager_1
          implementation_id: energy_grid
        - module_id: evse_manager_2
          implementation_id: energy_grid

  evse_manager_1:
    module: EvseManager
    config_module:
      connector_id: 1
      charge_mode: DC
    connections:
      bsp:
        - module_id: safety_bsp
          implementation_id: board_support
      powersupply_DC:
        - module_id: powersupply_dc
          implementation_id: main
      # ... remaining connections ...

  evse_manager_2:
    module: EvseManager
    config_module:
      connector_id: 2
      charge_mode: DC
    connections:
      bsp:
        - module_id: safety_bsp_2
          implementation_id: board_support
      powersupply_DC:
        - module_id: powersupply_dc        # Same power supply
          implementation_id: main           # shared DC bus
      # ... remaining connections ...

  ocpp:
    module: OCPP201
    connections:
      evse_manager:
        - module_id: evse_manager_1
          implementation_id: evse
        - module_id: evse_manager_2
          implementation_id: evse
      evse_energy_sink:
        - module_id: energy_manager
          implementation_id: main
          mapping:
            evse: 0         # Station-level
        - module_id: energy_manager
          implementation_id: main
          mapping:
            evse: 1         # EVSE 1
        - module_id: energy_manager
          implementation_id: main
          mapping:
            evse: 2         # EVSE 2
```

### 10.3 Complete Wiring Summary

```
                          Connection Map (Single-EVSE)

  EnergyManager.energy_trunk ──────────► evse_manager.energy_grid
  OCPP201.evse_energy_sink[0] ─────────► energy_manager.main (evse=0)
  OCPP201.evse_energy_sink[1] ─────────► energy_manager.main (evse=1)
  OCPP201.evse_manager ────────────────► evse_manager.evse
  EvseManager.powersupply_DC ──────────► powersupply_dc.main
  ThermalCoordinator.energy_manager ───► energy_manager.main (optional)
  HvacDriver.hvac_meter ───────────────► energy_manager external powermeter
```

## 11. Interaction with Charging Session Lifecycle

```
Session Phase        Energy Manager Action              Power Result
──────────────────── ─────────────────────────────────── ───────────────

IDLE                 enforce_limits: 0 W                 No modules active
(no EV connected)    (EVSE not requesting power)

EV PLUGS IN          EvseManager publishes initial        Still 0 W
(CP state B)         energy_flow_request (0 W)            (awaiting auth)

AUTHORIZED           EvseManager publishes                 Pre-charge starts
                     request with target V/I from EV

CABLE CHECK          enforce_limits: minimal               Single module
                     (only self-test current needed)        low voltage

PRE-CHARGE           enforce_limits: low current            Voltage ramp
                     (2A × target_V)

CHARGING             enforce_limits calculated:             Full power
                     min(EV_demand, site_budget,            (5 modules active)
                         OCPP_schedule, thermal_derate)

TAPER                EV reduces demand → EvseManager        Modules shed
(SoC 80%+)          publishes lower request →              to standby
                     EnergyManager may reallocate
                     surplus to EVSE 2 (if present)

SESSION END          enforce_limits: 0 W                    Ramp down
                     EvseManager initiates shutdown          Contactors open

FAULT                enforce_limits: 0 W immediately        Emergency disable
                     (thermal shutdown or safety fault)
```

## 12. Monitoring and Diagnostics

### 12.1 EverManager Logging

When `debug: true`, the EnergyManager logs the energy tree state every update cycle:

```
[EnergyManager] Tree update:
  GridNode: fuse=200000 W
    EVSE_1: request=150000 W, enforce=145000 W (OCPP: none, thermal: none, aux: 5000 W)
  External limits: none active
  HVAC power: 3200 W
```

### 12.2 OCPP Visibility

The CSMS can observe energy management effects through:

| OCPP Message | Data Source | Meaning |
|--------------|-------------|---------|
| `MeterValues` (Power.Active.Import) | PowerModuleDriver meter | Actual delivered power |
| `TransactionEvent` (meterValue) | PowerModuleDriver meter | Per-transaction energy |
| `GetCompositeSchedule` response | libocpp schedule engine | Effective merged schedule |
| `SetVariables` for custom vars | EnergyManager state | Available power, derate status |
| `NotifyEvent` | EvseManager limits | Limit changes logged as events |

### 12.3 HMI Display Integration

The EnergyManager state can be displayed on the user-facing HMI:

| Display Element | Source |
|----------------|--------|
| "Charging at X kW" | `enforce_limits.total_power_W` (actual allowed) |
| "Max available: Y kW" | Site budget minus aux loads |
| "Reduced power" indicator | `derate_request.derate_level != None` |
| "Grid limit active" | OCPP schedule is limiting |
| Estimated completion time | Based on `enforce_limits` schedule look-ahead |

## 13. Testing

### 13.1 Unit Tests

| Test | Validates |
|------|-----------|
| Single-EVSE full power | Grid limit > demand → full allocation |
| Single-EVSE OCPP limit | OCPP 100 kW < demand 150 kW → enforce 100 kW |
| Dual-EVSE equal split | Both EVSEs demand 150 kW, only 150 kW available → 75 kW each |
| Dual-EVSE asymmetric | EVSE 1 demands 30 kW, EVSE 2 demands 150 kW → 30 + 120 kW |
| Auxiliary load deduction | HVAC at 7 kW → available reduced by 7 kW |
| Thermal derate Moderate | Cabinet 42°C → enforce = 75% of site budget |
| Thermal derate Shutdown | Cabinet 52°C → enforce = 0 W, session terminates |
| Schedule time-series | OCPP schedule with 3 periods → enforce changes at boundaries |
| Min allocation floor | Both EVSEs active, one at minimum → 10 kW floor maintained |

### 13.2 SIL Configuration

```yaml
# config-sil-energy-manager.yaml
active_modules:

  energy_manager:
    module: EnergyManager
    config_module:
      update_interval: 1000
      debug: true
    connections:
      energy_trunk:
        - module_id: evse_manager_sim
          implementation_id: energy_grid

  evse_manager_sim:
    module: EvseManager
    config_module:
      charge_mode: DC
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

  safety_sim:
    module: SafetySupervisorSimulator
  power_sim:
    module: PowerModuleSimulator
```

### 13.3 Integration Test Scenarios

| Scenario | Steps | Expected Result |
|----------|-------|-----------------|
| OCPP demand response | CSMS sends 100 kW profile during 150 kW session | Output drops to 100 kW within 30 s |
| OCPP profile expiry | 100 kW profile expires | Output returns to full power |
| Dual-EVSE rebalance | EVSE 1 tapers while EVSE 2 ramps | Surplus redirected within 1 s |
| HVAC spike | HVAC power jumps from 3 kW to 7 kW | Available for EV reduced by 4 kW |
| Thermal + OCPP combined | Cabinet hot (derate 75%) + OCPP limit 120 kW | Enforce = min(site×0.75−aux, 120 kW) |
| Cold start, no CSMS | Boot without CSMS connection | Full local power, no OCPP limits |
| Grid connection upgrade | Operator sets higher fuse limit via CSMS | Available power increases immediately |

## 14. Related Documentation

- [[03 - EVerest OCPP201 Backend Integration]] — Smart charging profiles, composite schedules, `evse_energy_sink`
- [[02 - EVerest Power Module Driver]] — Power supply control, module shedding, current distribution
- [[04 - EVerest HVAC Driver]] — HVAC power metering, thermal derating signals, ThermalCoordinator
- [[01 - EVerest Safety Supervisor Integration]] — BSP module, contactor control, safety boundary
- [[01 - Software Framework]] — EVerest microservices architecture, MQTT IPC
- [[02 - Communication Protocols]] — CAN bus topology, OCPP network wiring
- [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]] — Interface contracts, `energy` interface specification
- [[docs/System/01 - System Architecture|01 - System Architecture]] — Control hierarchy, modular power architecture
