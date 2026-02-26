# Safety Supervisor Controller Software

Tags: #dcfc #safety #firmware #stm32 #software

Related: [[01 - Software Framework]] | [[research/01 - Safety Philosophy|01 - Safety Philosophy]] | [[docs/System/01 - System Architecture|01 - System Architecture]]

## 1. Overview

The Safety Supervisor Controller is a dedicated microcontroller (STM32Fxxx) running independent safety-critical firmware. It operates as the middle layer between the hardwired interlock chain and the CM5 main controller running Everest. Its sole purpose is to enforce safety invariants—even if the main controller hangs, crashes, or issues incorrect commands.

This controller is **not** part of the EVerest software stack. It runs bare-metal or minimal RTOS firmware with deterministic timing guarantees.

```
┌──────────────────────────────────────────────────────────────────┐
│                      SAFETY LAYERS                               │
│                                                                  │
│  LAYER 3: CM5 + EVerest (supervisory, non-safety)                │
│     │  Session logic, OCPP, ISO 15118, energy management         │
│     │  Sends "enable request" and setpoints                      │
│     ▼                                                            │
│  LAYER 2: SAFETY SUPERVISOR (STM32, SIL2/PLd)                    │
│     │  Validates all preconditions before allowing power         │
│     │  Monitors fault signals, watchdog on CM5                   │
│     │  Can override CM5 commands and force shutdown              │
│     ▼                                                            │
│  LAYER 1: HARDWARE INTERLOCK CHAIN (no software)                 │
│     E-STOP → Door → IMD → RCD → Thermal → Safety Relay           │
│     Hardwired series loop, trips regardless of any controller    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 2. Responsibilities

The safety supervisor handles functions that must continue operating correctly independent of the main controller:

| Function | Description |
|----------|-------------|
| **Contactor Sequencing** | Controls AC input, precharge, and DC output contactor timing |
| **Precharge Verification** | Confirms DC bus voltage reaches target before closing main contactors |
| **Welding Detection** | Checks contactor feedback after open command to detect welded contacts |
| **Watchdog on CM5** | Expects periodic heartbeat from CM5; forces shutdown on timeout |
| **Fault Aggregation** | Reads all safety input signals and determines appropriate response |
| **Voltage/Current Limits** | Independent OVP/OCP check using its own ADC readings |
| **Ground Fault Response** | Reacts to IMD and RCD fault signals |
| **Emergency Stop Handling** | Processes E-STOP input and coordinates safe de-energization |
| **State Reporting** | Reports safety state and fault codes to CM5 over CAN/UART |

## 3. Hardware Interface

### 3.1 Inputs

```
┌──────────────────────────────────────────────────────────────┐
│                SAFETY SUPERVISOR INPUTS                       │
│                                                              │
│  DIGITAL INPUTS (Isolated, Active-Low):                      │
│   DI0: E-STOP status (NC contact)                            │
│   DI1: Door interlock (NC contact)                           │
│   DI2: IMD OK/Fault relay                                    │
│   DI3: RCD trip relay                                        │
│   DI4: AC breaker AUX contact                                │
│   DI5: DC contactor AUX feedback (main)                      │
│   DI6: DC contactor AUX feedback (precharge)                 │
│   DI7: Connector latch status                                │
│   DI8: Thermal overload trip                                 │
│                                                              │
│  ANALOG INPUTS:                                              │
│   AI0: DC bus voltage (resistive divider, 0-1200V → 0-3.3V) │
│   AI1: DC output current (Hall sensor, 0-600A → 0-3.3V)     │
│   AI2: Heatsink temperature (NTC thermistor)                 │
│   AI3: Ambient temperature (NTC thermistor)                  │
│                                                              │
│  COMMUNICATION:                                              │
│   CAN: From CM5 (heartbeat, enable request, setpoints)       │
│   UART: Backup link to CM5 (optional, for diagnostics)       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 Outputs

```
┌──────────────────────────────────────────────────────────────┐
│                SAFETY SUPERVISOR OUTPUTS                      │
│                                                              │
│  DIGITAL OUTPUTS (Isolated, Fail-Safe = OFF):                │
│   DO0: AC input contactor coil drive                         │
│   DO1: Precharge contactor coil drive                        │
│   DO2: DC main contactor coil drive                          │
│   DO3: Outlet contactor coil drive (connector 1)             │
│   DO4: Outlet contactor coil drive (connector 2, if dual)    │
│   DO5: Power module ENABLE signal (to CAN #1 bus)            │
│   DO6: Fault LED / buzzer                                    │
│   DO7: Safety relay ENABLE (to hardware interlock chain)      │
│                                                              │
│  COMMUNICATION:                                              │
│   CAN: To CM5 (safety state, fault codes, measurements)      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## 4. Safety State Machine

The core of the safety supervisor firmware is a deterministic state machine. Transitions are driven by input conditions and commands from the CM5.

```
                        ┌─────────┐
                   ┌───▶│  INIT   │
                   │    └────┬────┘
                   │         │ Self-test pass
                   │         ▼
                   │    ┌─────────┐
                   │    │  IDLE   │◀──────────────────────────┐
                   │    └────┬────┘                           │
                   │         │ CM5 enable + all inputs OK     │
                   │         ▼                                │
                   │    ┌──────────┐                          │
                   │    │ AC_CLOSE │ Close AC contactor       │
                   │    └────┬─────┘                          │
                   │         │ AC contactor feedback OK       │
                   │         ▼                                │
                   │    ┌───────────┐                         │
                   │    │ PRECHARGE │ Close precharge relay   │
                   │    └────┬──────┘                         │
                   │         │ DC bus voltage within target   │
                   │         │ (timeout → FAULT)              │
                   │         ▼                                │
                   │    ┌───────────┐                         │
                   │    │ DC_CLOSE  │ Close DC main contactor │
                   │    └────┬──────┘                         │
                   │         │ DC contactor feedback OK       │
                   │         ▼                                │
                   │    ┌───────────┐                         │
                   │    │ CHARGING  │ Power delivery active   │
                   │    └────┬──────┘                         │
                   │         │ CM5 stop or session end        │
                   │         ▼                                │
                   │    ┌───────────┐                         │
                   │    │ SHUTDOWN  │ Orderly de-energization │───┘
                   │    └───────────┘
                   │
                   │    ┌───────────┐
                   └────│   FAULT   │◀─── Any fault from any state
                        └────┬──────┘
                             │ Fault cleared + CM5 reset cmd
                             ▼
                        ┌───────────┐
                        │   IDLE    │
                        └───────────┘

                        ┌───────────┐
                        │ EMERGENCY │◀─── E-STOP from any state
                        │   STOP    │     (immediate contactor drop)
                        └───────────┘
                             │ E-STOP released + CM5 reset
                             ▼
                        ┌───────────┐
                        │   INIT    │ (full re-initialization)
                        └───────────┘
```

### 4.1 State Descriptions

| State | Entry Condition | Actions | Exit Condition |
|-------|----------------|---------|----------------|
| **INIT** | Power-on / E-STOP recovery | Self-test: check ADC, CAN, GPIO. Verify all outputs OFF. | Self-test pass → IDLE |
| **IDLE** | Init complete or shutdown complete | All contactors open. Monitor inputs. Wait for CM5 enable. | CM5 enable + all OK → AC_CLOSE |
| **AC_CLOSE** | Enable received, inputs healthy | Energize AC contactor. Start feedback timeout (200 ms). | Feedback OK → PRECHARGE |
| **PRECHARGE** | AC contactor confirmed closed | Close precharge relay. Monitor DC bus voltage ramp. | V_bus within target ±5% → DC_CLOSE |
| **DC_CLOSE** | Precharge complete | Open precharge relay, close DC main contactor. Verify feedback. | Feedback OK → CHARGING |
| **CHARGING** | DC contactor confirmed closed | Monitor V/I limits, watchdog, all safety inputs. Assert ENABLE to power modules. | CM5 stop → SHUTDOWN |
| **SHUTDOWN** | Normal stop requested | De-assert ENABLE. Wait for current < 5A. Open DC contactor, then AC contactor. | All contactors open → IDLE |
| **FAULT** | Any monitored fault | Immediate: disable power modules. Sequential: open DC, then AC contactors. Latch fault code. | Fault cleared + CM5 reset → IDLE |
| **EMERGENCY STOP** | E-STOP input active | Drop all contactors immediately (no sequencing). | E-STOP released + CM5 reset → INIT |

### 4.2 Fault Conditions

| Fault Code | Condition | Response Time | Severity |
|------------|-----------|---------------|----------|
| `F01` | DC overvoltage (>1050V) | < 1 ms (hardware comparator) | Critical |
| `F02` | DC overcurrent (>110% rated) | < 10 ms | Critical |
| `F03` | IMD fault (insulation loss) | < 100 ms | Critical |
| `F04` | RCD trip (ground fault) | < 30 ms | Critical |
| `F05` | Contactor weld detected | At open command | Critical |
| `F06` | Precharge timeout (>3 s) | 3 s | Major |
| `F07` | CM5 watchdog timeout | 2 s | Major |
| `F08` | Over-temperature (component) | 500 ms | Major |
| `F09` | Door interlock open | < 100 ms | Major |
| `F10` | Contactor feedback mismatch | 200 ms | Major |
| `F11` | CAN bus communication loss | 1 s | Minor |
| `F12` | ADC self-test failure | At INIT | Critical |

## 5. Firmware Architecture

### 5.1 Software Structure

```
┌──────────────────────────────────────────────────────────┐
│                  FIRMWARE LAYERS                          │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              APPLICATION LAYER                     │  │
│  │  ┌──────────────┐  ┌──────────────┐                │  │
│  │  │ Safety State │  │   Fault      │                │  │
│  │  │   Machine    │  │  Manager     │                │  │
│  │  └──────────────┘  └──────────────┘                │  │
│  │  ┌──────────────┐  ┌──────────────┐                │  │
│  │  │  Contactor   │  │  Measurement │                │  │
│  │  │  Sequencer   │  │  Validator   │                │  │
│  │  └──────────────┘  └──────────────┘                │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              DRIVER LAYER                          │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐  │  │
│  │  │  GPIO  │ │  ADC   │ │  CAN   │ │  Watchdog  │  │  │
│  │  │ Driver │ │ Driver │ │ Driver │ │  Timer     │  │  │
│  │  └────────┘ └────────┘ └────────┘ └────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              HAL (STM32 LL/HAL)                    │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │         BARE-METAL / FreeRTOS (optional)           │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 5.2 Task Scheduling

The firmware uses a strict cyclic execution model for deterministic timing:

| Task | Period | Deadline | Priority |
|------|--------|----------|----------|
| Safety Input Scan | 1 ms | 1 ms | Highest |
| ADC Measurement + OVP/OCP | 1 ms | 1 ms | Highest |
| State Machine Tick | 10 ms | 10 ms | High |
| Contactor Sequencer | 10 ms | 10 ms | High |
| CM5 Watchdog Check | 100 ms | 100 ms | Medium |
| CAN TX (status report) | 100 ms | 100 ms | Medium |
| CAN RX (command parse) | 10 ms | 10 ms | High |
| Self-Test (background) | 1 s | 1 s | Low |

### 5.3 Key Design Principles

- **No dynamic memory allocation** — all buffers statically allocated at compile time
- **No blocking calls** — all I/O is interrupt-driven or polled within cycle time
- **Defensive coding** — all inputs range-checked, all state transitions validated
- **Fail-safe defaults** — all outputs default to OFF/open on any unexpected condition
- **Independent measurement** — OVP/OCP uses STM32's own ADC, not values reported by CM5 or power modules
- **Watchdog stack** — internal IWDG (independent watchdog) resets the STM32 if firmware hangs; CM5 heartbeat timeout triggers safe shutdown if CM5 hangs

## 6. CM5 Communication Protocol

### 6.1 CAN Interface

Communication between the safety supervisor and CM5 uses CAN 2.0B at 500 kbps.

#### Messages from CM5 → Safety Supervisor

| CAN ID | Name | Period | Payload |
|--------|------|--------|---------|
| `0x100` | Heartbeat | 500 ms | Counter (8-bit rolling), CM5 state |
| `0x101` | Enable Request | On change | Enable/Disable flag, requested V/I limits |
| `0x102` | Reset Command | On demand | Fault reset request, restart type |

#### Messages from Safety Supervisor → CM5

| CAN ID | Name | Period | Payload |
|--------|------|--------|---------|
| `0x200` | Safety Status | 100 ms | State, active faults (bitmask), contactor status |
| `0x201` | Measurements | 100 ms | DC bus voltage, DC current, temperatures |
| `0x202` | Fault Detail | On event | Fault code, timestamp, measured value at fault |

### 6.2 Heartbeat Protocol

```
CM5 sends 0x100 every 500 ms:
  Byte 0: Rolling counter (0-255, incrementing)
  Byte 1: CM5 operating state

Safety Supervisor checks:
  - Message received within 2 s window (4 missed heartbeats)
  - Counter is incrementing (not stuck)
  - If either fails → F07 (CM5 watchdog timeout) → FAULT state
```

### 6.3 Enable Handshake

The CM5 cannot directly control contactors. It can only *request* that the safety supervisor enable charging:

```
1. CM5 sends Enable Request (0x101) with desired V/I limits
2. Safety supervisor validates:
   - All safety inputs OK
   - Requested V/I within hardware limits
   - No active faults
   - Heartbeat active
3. If valid → begin contactor sequence (AC_CLOSE → PRECHARGE → ...)
4. If invalid → reject, report reason in Safety Status (0x200)
```

This ensures the CM5 never has direct authority over high-voltage contactors.

## 7. Contactor Sequencing Detail

### 7.1 Startup Sequence

```
Time ──────────────────────────────────────────────────────▶

AC Contactor:    ────┐
                     └─────────────────────────────────────
                     t0     Feedback check (200ms window)

Precharge Relay:          ────┐
                              └──────────────┐
                                             │ Open after
                              DC bus ramp     │ DC close

DC Bus Voltage:           ___/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
                              0V → target (±5%)

DC Contactor:                               ────┐
                                                 └─────────
                                                 Feedback OK

ENABLE to Bricks:                                     ────┐
                                                          └
                                                 Power flow

Typical timing:
  t0 = 0 ms:     AC contactor close command
  t0 + 200 ms:   AC feedback verified
  t0 + 250 ms:   Precharge relay close
  t0 + 1-3 s:    DC bus charged (depends on capacitance)
  t0 + 3 s:      Precharge open, DC contactor close
  t0 + 3.2 s:    DC feedback verified, ENABLE asserted
```

### 7.2 Shutdown Sequence

```
1. De-assert ENABLE to power modules
2. Wait for output current < 5A (max 2 s, else force)
3. Open DC main contactor
4. Verify DC contactor feedback (open confirmed)
5. Open AC input contactor
6. Verify AC contactor feedback
7. Transition to IDLE
```

### 7.3 Emergency Shutdown (E-STOP or Critical Fault)

```
1. Immediately de-assert all contactor outputs (DO0-DO4 = OFF)
2. De-assert ENABLE (DO5 = OFF)
3. De-assert safety relay ENABLE (DO7 = OFF)
   → Hardware interlock chain also drops all contactors
4. Latch fault code
5. Report to CM5 via CAN
```

No sequencing is performed during emergency shutdown—all outputs are dropped simultaneously within one scan cycle (1 ms).

## 8. Self-Test and Diagnostics

### 8.1 Power-On Self-Test (POST)

Executed during the INIT state before transitioning to IDLE:

| Test | Method | Pass Criteria |
|------|--------|---------------|
| ADC calibration | Read internal reference voltage | Within ±2% of expected |
| GPIO output test | Toggle each output, read back | Output matches command |
| Contactor feedback | Verify all contactors report OPEN | All DI feedback = OPEN |
| CAN loopback | Send/receive test frame | Frame received correctly |
| Watchdog test | Verify IWDG resets if not fed | Reset occurs within window |
| Flash CRC | Compare stored CRC against computed | Match |
| RAM pattern test | Write/read test patterns | All patterns verified |

### 8.2 Runtime Diagnostics

| Check | Period | Action on Failure |
|-------|--------|-------------------|
| ADC reference drift | 1 s | F12 → FAULT |
| Contactor feedback consistency | Every state transition | F10 → FAULT |
| CAN bus error counter | 100 ms | F11 (warning at 50%, fault at bus-off) |
| Stack overflow detection | 10 ms | Immediate reset via IWDG |
| Execution time monitoring | Every cycle | Warning if >80% of deadline |

## 9. Standards and Compliance

The safety supervisor firmware is developed according to:

| Standard | Scope | Target Level |
|----------|-------|-------------|
| IEC 61508 | Functional safety of E/E/PE systems | SIL 2 |
| ISO 13849 | Safety of machinery control systems | Performance Level d (PLd) |
| IEC 61851-1 | EV charging safety requirements | Clause 6 (safety functions) |
| IEC 62477-1 | Safety of power electronics | General requirements |
| MISRA C:2012 | Coding standard for safety-critical C | Required subset |

### 9.1 Development Practices for SIL 2

- Static analysis (MISRA C compliance) on every build
- Unit tests with branch coverage >90%
- Formal code review for all safety-critical paths
- Requirements traceability (requirement → code → test)
- Configuration management with signed firmware images
- Hardware fault tolerance: dual-channel inputs for critical signals (E-STOP, IMD)

## 10. Related Documentation

- [[01 - Software Framework]] — EVerest framework running on CM5
- [[research/01 - Safety Philosophy|01 - Safety Philosophy]] — Hardware interlock design
- [[research/02 - CM5 based Main Controller|02 - CM5 based Main Controller]] — CM5 architecture and EVSE aux board
- [[docs/System/01 - System Architecture|01 - System Architecture]] — System-level safety architecture
- [[docs/Hardware/01 - Hardware Components|01 - Hardware Components]] — Protection devices and contactors
