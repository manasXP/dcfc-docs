# ISO 15118 Vehicle Communication

Tags: #dcfc #iso15118 #everest #software #plc #slac #pnc #din70121

Related: [[03 - EVerest OCPP201 Backend Integration]] | [[01 - EVerest Safety Supervisor Integration]] | [[01 - Software Framework]] | [[02 - Communication Protocols]] | [[docs/System/03 - Standards Compliance|03 - Standards Compliance]]

## 1. Overview

ISO 15118 defines the high-level communication (HLC) between the EV and the EVSE during a DC fast charging session. It runs over PowerLine Communication (PLC) on the Control Pilot wire, enabling features beyond basic IEC 61851 signaling: negotiated charge parameters, Plug & Charge (PnC), scheduled charging, and bidirectional power transfer (V2G).

This note covers the protocol stack, DC charging message sequence, EVerest module implementation, and hardware requirements for the DCFC design.

```
┌─────────────────────────────────────────────────────────────────┐
│                     PROTOCOL STACK                               │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Application Layer                                        │  │
│  │  ISO 15118-2 / ISO 15118-20 / DIN SPEC 70121             │  │
│  │  (EXI-encoded XML messages)                               │  │
│  └─────────────────────────┬─────────────────────────────────┘  │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────────────┐  │
│  │  V2GTP (Vehicle-to-Grid Transfer Protocol)                │  │
│  │  8-byte header + EXI payload                              │  │
│  └─────────────────────────┬─────────────────────────────────┘  │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────────────┐  │
│  │  TLS 1.2 (ISO 15118-2 PnC) / TLS 1.3 (ISO 15118-20)     │  │
│  │  (optional for EIM, mandatory for PnC)                    │  │
│  └─────────────────────────┬─────────────────────────────────┘  │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────────────┐  │
│  │  TCP / IPv6 (link-local addresses, fe80::/10)             │  │
│  └─────────────────────────┬─────────────────────────────────┘  │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────────────┐  │
│  │  SLAC (Signal Level Attenuation Characterization)         │  │
│  │  ISO 15118-3 — EV-to-EVSE pairing over PLC               │  │
│  └─────────────────────────┬─────────────────────────────────┘  │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────────────┐  │
│  │  HomePlug Green PHY (HPGP)                                │  │
│  │  OFDM modulation, 1.8–30 MHz, ~10 Mbps                   │  │
│  └─────────────────────────┬─────────────────────────────────┘  │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────────────┐  │
│  │  Physical Layer: Control Pilot (CP) wire ↔ PE             │  │
│  │  Coupling transformer + TVS protection                    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Physical Layer: PLC over Control Pilot

### 2.1 HomePlug Green PHY

ISO 15118-3 specifies HomePlug Green PHY (HPGP) as the data link and physical layer. The PLC signal is modulated onto the **Control Pilot (CP) wire** referenced against **Protective Earth (PE)** inside the CCS charging cable.

| Parameter | Value |
|-----------|-------|
| Modulation | OFDM (Orthogonal Frequency Division Multiplexing) |
| Frequency band | 1.8–30 MHz |
| Data rate | ~10 Mbps (sufficient for V2G application traffic) |
| Max PSD at EV socket | -73 dBm/Hz (target -75 dBm/Hz per ISO 15118-3) |
| Coexistence | Operates simultaneously with 1 kHz IEC 61851 CP PWM signal |

### 2.2 CP Line Coupling Circuit

The PLC modem connects to the CP line through a coupling circuit:

```
PLC Modem                                    CP Line to Vehicle
  TX/RX ──► ┌──────────────────┐             ┌───────────────────┐
             │ 1:1:1 Coupling   │    CP ──────│   CCS Connector   │
             │ Transformer      ├─────────────│   (CP pin)        │
             │ (YT-61083-1)     │             │                   │
  GND ──────│                  │    PE ──────│   (PE pin)        │
             └──────────────────┘             └───────────────────┘
                    │
               ┌────┴────┐
               │ 3V TVS  │ ≤3 pF junction capacitance
               │ (PLC    │ (surge protection on PLC path)
               │  path)  │
               └────┬────┘
                    │
                   GND

On CP PWM circuit:
  12V TVS diode, ≤10 pF
  (protects PWM generator from PLC transients)
```

The coupling transformer provides:
- Galvanic isolation between PLC modem and CP line
- Impedance matching for HPGP OFDM signals
- High impedance at 1 kHz (so the PWM circuit is not affected)
- The 1:1:1 ratio (vs 1:5:4 for mains PLC) accounts for the short cable length (3–5 m)

### 2.3 PLC Modem Hardware

| Chip | Manufacturer | Host Interface | Grade | Notes |
|------|-------------|----------------|-------|-------|
| QCA7000 / QCA7005 | Qualcomm | SPI (Mode 3, ≤12 MHz) | QCA7005: automotive | Linux kernel driver upstream (`CONFIG_QCA7000_SPI`). Appears as standard netdev. |
| IS32CG5317 | Lumissil (ISSI) | SPI or RMII (100 Mbps Ethernet) | AEC-Q100 Grade 2 | EVerest provides out-of-the-box support. BMW production use. |

**How the PLC modem connects to the Phytec SBC:**

```
Option A: SPI (QCA7005 or IS32CG5317 in SPI mode)

  Phytec SBC SoC ─── SPI bus ──► QCA7005 chip ──► Coupling transformer ──► CP line
                           │
                           └─ Linux sees as: eth1 (netdev via qca7000 driver)

Option B: RMII / Ethernet (IS32CG5317 in RMII mode)

  Phytec SBC SoC ─── RMII/MII ──► IS32CG5317 ──► Coupling transformer ──► CP line
                            │
                            └─ Linux sees as: eth1 (standard Ethernet MAC)

Either way, both EvseSlac and EvseV2G point to the same interface name (e.g., "cb_plc").
```

> [!note] PHYTEC PEB-X-005
> PHYTEC's EVSE expansion board for the AM62x uses the IS32CG5317 over SPI. It includes a TI MSPM0 MCU for real-time EVSE logic. Suitable as an evaluation platform alongside the Phytec SBC.

## 3. SLAC: EV-to-EVSE Pairing

SLAC (Signal Level Attenuation Characterization) is the mechanism by which the EV determines which physical EVSE it is connected to. This prevents cross-matching when multiple chargers share a PLC segment at a charging plaza.

### 3.1 SLAC Matching Process

```
EV (EVCC)                                        EVSE (SECC)
    │                                                 │
    │  1. CP state A → B (EV plugs in)                │
    │                                                 │ EvseManager calls
    │                                                 │ EvseSlac.enter_bcd()
    │                                                 │
    │──── CM_SLAC_PARM.REQ (broadcast) ──────────────▶│
    │     EV MAC address, run ID                      │
    │                                                 │
    │◀─── CM_SLAC_PARM.CNF ──────────────────────────│
    │     Number of sounds requested (10)             │
    │                                                 │
    │──── CM_MNBC_SOUND.IND ×10 (sounding) ─────────▶│
    │     EV transmits 10 sound frames                │ EVSE measures
    │     (all EVSEs on segment listen)               │ attenuation per
    │                                                 │ OFDM tone group
    │                                                 │
    │◀─── CM_ATTEN_CHAR.IND ─────────────────────────│
    │     Average attenuation across 58 groups        │
    │     (each EVSE on segment reports)              │
    │                                                 │
    │     EV selects EVSE with LOWEST attenuation     │
    │     (= physically closest = connected)          │
    │                                                 │
    │──── CM_SLAC_MATCH.REQ ─────────────────────────▶│
    │     Match request to selected EVSE              │
    │                                                 │
    │◀─── CM_SLAC_MATCH.CNF ─────────────────────────│
    │     NMK (128-bit AES key)                       │
    │     NID (54-bit network ID)                     │
    │                                                 │
    │     EV joins EVSE's encrypted PLC network       │
    │                                                 │
    │◀─── EvseSlac publishes dlink_ready(true) ──────│
    │                                                 │
    │     V2G TCP/IPv6 connection can now begin        │
```

### 3.2 Key SLAC Properties

| Property | Value |
|----------|-------|
| Sounding count | 10 frames (per ISO 15118-3 minimum) |
| Matching criteria | Lowest average signal attenuation |
| Encryption | AES-CBC-128 (NMK exchanged in MATCH.CNF) |
| Network isolation | Each matched EV-EVSE pair gets a unique NMK/NID |
| Retry behavior | If SLAC fails, EV restarts from PARM.REQ (EvseSlac handles via `reset_instead_of_fail: true`) |

## 4. ISO 15118-2 DC Charging Message Sequence

### 4.1 Protocol Negotiation and Session Setup

| # | Message | Direction | Key Parameters |
|---|---------|-----------|----------------|
| 1 | **SDP Request/Response** | EV → EVSE (UDP 15118) | Discover EVSE IPv6 address, TCP port, TLS flag |
| 2 | TCP/TLS Handshake | Bidirectional | Certificate exchange (PnC only) |
| 3 | **SupportedAppProtocolReq** | EV → EVSE | Supported protocol namespaces (DIN 70121, ISO 15118-2) with priority |
| 3r | **SupportedAppProtocolRes** | EVSE → EV | Selected protocol, `OK_SuccessfulNegotiation` |
| 4 | **SessionSetupReq** | EV → EVSE | `EVCCID` (EV MAC address, 6-byte hex) |
| 4r | **SessionSetupRes** | EVSE → EV | `SessionID` (8-byte unique), `EVSEID` (e.g., `DE*PNX*E12345*1`) |

### 4.2 Service Discovery and Authorization

| # | Message | Direction | Key Parameters |
|---|---------|-----------|----------------|
| 5 | **ServiceDiscoveryReq** | EV → EVSE | Optional service scope filter |
| 5r | **ServiceDiscoveryRes** | EVSE → EV | `ChargeService` with `EnergyTransferMode` (DC_extended), optional `ValueAddedServiceList` |
| 6 | **PaymentServiceSelectionReq** | EV → EVSE | `SelectedPaymentOption`: `ExternalPayment` (EIM) or `Contract` (PnC) |
| 6r | **PaymentServiceSelectionRes** | EVSE → EV | `ResponseCode` |
| 7 | **PaymentDetailsReq** (PnC) | EV → EVSE | `eMAID`, `ContractSignatureCertChain` |
| 7r | **PaymentDetailsRes** (PnC) | EVSE → EV | `GenChallenge` (16-byte nonce) |
| 8 | **AuthorizationReq** | EV → EVSE | `GenChallenge` echoed (PnC) or empty (EIM) |
| 8r | **AuthorizationRes** | EVSE → EV | `EVSEProcessing`: `Ongoing` (EVSE validating via OCPP) or `Finished` |

> [!tip] Auth flow integration
> For **EIM** (RFID): EvseV2G publishes `require_auth_eim` → Auth module validates token via OCPP → `authorization_response(Accepted)` → AuthorizationRes(Finished).
> For **PnC**: EvseV2G publishes `require_auth_pnc` with contract cert → OCPP sends `Authorize.req` with `iso15118CertificateHashData` → CSMS validates → `authorization_response(Accepted)`.
> See [[03 - EVerest OCPP201 Backend Integration#7. Auth Module and Token Validation Flow]] for the full flow.

### 4.3 DC Charge Parameter Exchange

| # | Message | Direction | Key Parameters |
|---|---------|-----------|----------------|
| 9 | **ChargeParameterDiscoveryReq** | EV → EVSE | `RequestedEnergyTransferMode`: `DC_extended` |

**EV sends `DC_EVChargeParameter`:**

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `EVMaximumCurrentLimit` | A | 200 A | Max current EV can accept |
| `EVMaximumVoltageLimit` | V | 800 V | Max voltage EV can accept |
| `EVMaximumPowerLimit` | W | 150000 W | Max power (optional) |
| `EVEnergyCapacity` | Wh | 77000 Wh | Battery capacity (optional) |
| `EVEnergyRequest` | Wh | 60000 Wh | Energy requested (optional) |
| `FullSOC` | % | 100 | SoC considered fully charged |
| `BulkSOC` | % | 80 | SoC considered bulk complete |

**EVSE responds with `DC_EVSEChargeParameter`:**

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `EVSEMaximumCurrentLimit` | A | 187.5 A | Max current EVSE can deliver |
| `EVSEMaximumVoltageLimit` | V | 1000 V | Max voltage |
| `EVSEMaximumPowerLimit` | W | 150000 W | Max power |
| `EVSEMinimumCurrentLimit` | A | 0 A | Min current |
| `EVSEMinimumVoltageLimit` | V | 200 V | Min voltage |
| `EVSECurrentRegulationTolerance` | A | 2 A | Current regulation accuracy |
| `EVSEPeakCurrentRipple` | A | 5 A | Output current ripple |
| `SAScheduleList` | — | — | Charging schedules (time-based power limits) |

### 4.4 Cable Check (Isolation Test)

| # | Message | Direction | Key Parameters |
|---|---------|-----------|----------------|
| 10 | **CableCheckReq** (loop) | EV → EVSE | `DC_EVStatus`: `EVReady`, `EVErrorCode`, `EVRESSSOC` (SoC %) |
| 10r | **CableCheckRes** (loop) | EVSE → EV | `EVSEIsolationStatus`: Valid/Warning/Fault, `EVSEProcessing`: Ongoing/Finished |

During CableCheck, the EVSE closes the DC output contactor at low voltage and runs the IMD self-test. EvseManager calls `start_cable_check` → IMD module tests insulation resistance (≥500 Ω/V) → `cable_check_finished(true)` → `update_isolation_status(Valid)` → CableCheckRes with `EVSEProcessing: Finished`.

### 4.5 Pre-Charge

| # | Message | Direction | Key Parameters |
|---|---------|-----------|----------------|
| 11 | **PreChargeReq** (loop) | EV → EVSE | `EVTargetVoltage` (V), `EVTargetCurrent` (1-2 A inrush limit) |
| 11r | **PreChargeRes** (loop) | EVSE → EV | `EVSEPresentVoltage` (actual output voltage) |

The EVSE ramps DC output toward `EVTargetVoltage`. Loop continues until `|EVSEPresentVoltage − EVTargetVoltage| < 20 V`. Once converged, the EV safely closes its main battery contactors (no arcing due to matched voltage).

```
DC Bus Voltage
    ▲
    │         EVTargetVoltage (e.g., 400V)
    │    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
    │                               /‾‾‾‾‾‾‾‾‾‾‾
    │                         /‾‾‾‾‾
    │                    /‾‾‾‾    EVSEPresentVoltage ramps up
    │               /‾‾‾‾
    │          /‾‾‾‾       ←──── ΔV < 20V: EV closes contactors
    │     /‾‾‾‾
    │ /‾‾‾
    └──────────────────────────────────────────────▶ Time
         PreCharge loop (~60ms per exchange)
```

### 4.6 Charging: CurrentDemand Loop

| # | Message | Direction | Key Parameters |
|---|---------|-----------|----------------|
| 12 | **PowerDeliveryReq** (Start) | EV → EVSE | `ChargeProgress: Start`, selected schedule |
| 13 | **CurrentDemandReq** (~60 ms loop) | EV → EVSE | See table below |
| 13r | **CurrentDemandRes** (~60 ms loop) | EVSE → EV | See table below |

**EV sends in CurrentDemandReq:**

| Field | Description |
|-------|-------------|
| `EVTargetCurrent` (A) | Primary control variable — current the EV requests |
| `EVTargetVoltage` (V) | Target voltage |
| `EVMaximumCurrentLimit` (A) | May change during session (thermal derating) |
| `EVMaximumVoltageLimit` (V) | Upper voltage clamp |
| `EVMaximumPowerLimit` (W) | Power cap |
| `EVRESSSOC` (%) | Current state of charge |
| `BulkChargingComplete` | True when SoC ≥ BulkSOC |
| `ChargingComplete` | True when SoC ≥ FullSOC → triggers session end |

**EVSE responds in CurrentDemandRes:**

| Field | Description |
|-------|-------------|
| `EVSEPresentVoltage` (V) | Measured DC output voltage |
| `EVSEPresentCurrent` (A) | Measured DC output current |
| `EVSEMaximumCurrentLimit` (A) | May change (module fault, thermal derating, smart charging) |
| `EVSEMaximumVoltageLimit` (V) | May change |
| `EVSEMaximumPowerLimit` (W) | May change (OCPP smart charging profile) |
| `EVSECurrentLimitAchieved` (bool) | True if delivering at max current |
| `EVSEVoltageLimitAchieved` (bool) | True if at voltage limit |
| `EVSEPowerLimitAchieved` (bool) | True if at power limit |

> [!important] Control authority
> The **EV is the current regulator** — it requests a target current, the EVSE delivers up to that limit. The EVSE reports its actual output and maximum limits. If the EVSE's max drops (e.g., module fault, smart charging), the EV must reduce its request accordingly.
>
> The `EVSEPresentVoltage` and `EVSEPresentCurrent` values come from the [[02 - EVerest Power Module Driver]] via `update_dc_present_values`, and the `EVSEMaximum*` limits come from `update_dc_maximum_limits`, which tracks the [[04 - Power Module CAN Bus Interface|power module]] aggregate capabilities.

### 4.7 Session Termination

| # | Message | Direction | Key Parameters |
|---|---------|-----------|----------------|
| 14 | **PowerDeliveryReq** (Stop) | EV → EVSE | `ChargeProgress: Stop` or `Renegotiate` |
| 15 | **WeldingDetectionReq** (optional) | EV → EVSE | `DC_EVStatus` |
| 15r | **WeldingDetectionRes** | EVSE → EV | `EVSEPresentVoltage` (confirms safe voltage after contactor open) |
| 16 | **SessionStopReq** | EV → EVSE | `Action`: `Terminate` or `Pause` |
| 16r | **SessionStopRes** | EVSE → EV | `ResponseCode` |

If `Pause`: session parameters are stored, `SessionID` retained for resumption.
If `Terminate`: all resources freed, connector unlocked.

### 4.8 Complete DC Session Timeline

```
Layer        CP/SLAC              ISO 15118                        EVSE Hardware
──────────── ──────────────────── ──────────────────────────────── ─────────────────

EV Plugs In  CP: A → B
             5% PWM on CP        ┐
             SLAC matching       │ SDP → TCP → SupportedAppProto
             dlink_ready(true)   │ SessionSetup
                                 │ ServiceDiscovery
                                 │ PaymentServiceSelection
                                 │ Authorization (EIM/PnC)        RFID / OCPP auth
                                 │ ChargeParameterDiscovery       Report capabilities
                                 ├─────────────────────────────── ──────────────────
                                 │ CableCheck loop                IMD self-test
                                 │   EVSEIsolationStatus=Valid    Contactors closed
                                 ├─────────────────────────────── ──────────────────
                                 │ PreCharge loop                 Voltage ramp
                                 │   V_out → V_target (ΔV<20V)   Single module
                                 │   EV closes its contactors
                                 ├─────────────────────────────── ──────────────────
                                 │ PowerDelivery(Start)
                                 │ CurrentDemand loop (~60ms)     All modules
                                 │   EV requests I, V             delivering power
                                 │   EVSE reports actual V, I
                                 │   EVSE reports SoC on display
                                 │   ...minutes to hours...
                                 │   ChargingComplete = true
                                 ├─────────────────────────────── ──────────────────
                                 │ PowerDelivery(Stop)            Ramp current → 0
                                 │ WeldingDetection (optional)    Open DC contactor
                                 │ SessionStop(Terminate)         Open AC contactor
                                 └─────────────────────────────── ──────────────────
             CP: B → A (unplug)                                   Unlock connector
```

## 5. DIN SPEC 70121 (Legacy DC Protocol)

DIN SPEC 70121 is a German pre-standard (2012/2014) used by early CCS chargers before ISO 15118-2 was finalized. Many 2013–2018 EVs only support DIN 70121.

| Aspect | DIN SPEC 70121 | ISO 15118-2 |
|--------|---------------|-------------|
| Physical layer | HomePlug GP / SLAC (same) | Same |
| Encoding | EXI (same) | Same |
| Charging modes | DC only | AC and DC |
| TLS / Security | Not supported (plain TCP) | TLS 1.2 optional (mandatory for PnC) |
| Plug & Charge | Not supported | Supported |
| Smart charging schedules | Not supported | Supported (SAScheduleList) |
| V2G bidirectional | Not supported | Not natively (extension exists) |
| Namespace | `urn:din:70121:2012:MsgDef` | `urn:iso:15118:2:2013:MsgDef` |
| Protocol selection | Via `SupportedAppProtocolReq` negotiation | Same mechanism |

The EVSE must support both protocols. EvseV2G's `supported_DIN70121: true` (default) enables this. When an EV only advertises a DIN namespace in `SupportedAppProtocolReq`, EvseV2G negotiates down to DIN 70121 automatically. The DC charging message sequence is essentially the same (CableCheck, PreCharge, CurrentDemand, etc.) but with a DIN-specific namespace and no support for TLS, PnC, or schedules.

> [!warning] Backward compatibility
> Both DIN 70121 and ISO 15118-2 must be supported for maximum EV compatibility. Disabling DIN support will prevent charging for older BMWs, early Teslas (in CCS markets), and early VW Group vehicles.

## 6. ISO 15118-20 (Next Generation)

ISO 15118-20 (2022) is a major revision adding new capabilities.

### 6.1 Key Differences from ISO 15118-2

| Feature | ISO 15118-2 | ISO 15118-20 |
|---------|-------------|--------------|
| TLS version | TLS 1.2 | TLS 1.3 (mandatory) |
| Bidirectional (V2G/V2H) | Not native | Native BPT support |
| Charging control modes | Scheduled only | Scheduled + **Dynamic** mode |
| DC main loop message | `CurrentDemandReq/Res` | `DC_ChargeLoopReq/Res` |
| Schedule negotiation | Embedded in ChargeParameterDiscovery | Separate `ScheduleExchangeReq/Res` |
| Contract certificates | Single certificate | Multiple simultaneous certificates |
| Cipher suites | Various TLS 1.2 suites | `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256` |

### 6.2 Scheduled vs Dynamic Mode

**Scheduled Mode** (similar to ISO 15118-2):
- EVSE provides power schedules (time windows with limits)
- EV selects a schedule, can request renegotiation
- Uses new `ScheduleExchangeReq/Res` message

**Dynamic Mode** (new):
- EV yields control authority to the EVSE
- EVSE provides single power setpoints without multi-step schedule negotiation
- Better for grid-responsive and smart charging
- EVerest exposes via `DcChargeDynamicModeValues` type

### 6.3 Bidirectional Power Transfer (BPT)

ISO 15118-20 natively defines V2G and V2H with new `DC_ChargeLoopReq` parameters:
- `EVMaximumDischargePower` / `EVMinimumDischargePower`
- `EVMaximumDischargeCurrent` / `EVMinimumDischargeCurrent`
- Separate energy tracking for discharge in meter readings

EVerest supports BPT via the `bpt_setup` command on the `ISO15118_charger` interface.

### 6.4 EVerest ISO 15118-20 Status

EVerest has `libiso15118` (C++ library) under active development for 15118-20 DC support, with state machines for all DC phases. A simulation config (`config-sil-dc-d20.yaml`) is available for testing.

## 7. EVerest Module Architecture

### 7.1 Module Relationships

```
┌───────────────────────────────────────────────────────────────────────┐
│                          Phytec SBC (Linux + EVerest)                         │
│                                                                       │
│  ┌──────────────┐                                                     │
│  │  EvseManager │                                                     │
│  │              │─── slac: ──────────────► ┌────────────┐             │
│  │  charge_mode │                          │  EvseSlac  │             │
│  │  = DC        │                          │            │             │
│  │              │                          │ device:    │             │
│  │              │                          │  cb_plc    │             │
│  │              │─── hlc: ───────────────► └────────────┘             │
│  │              │                          ┌────────────┐             │
│  │              │                          │  EvseV2G   │             │
│  │              │                          │            │             │
│  │              │                          │ provides:  │             │
│  │              │                          │  charger   │             │
│  │              │                          │  (ISO15118 │             │
│  │              │                          │  _charger) │             │
│  │              │                          │            │             │
│  │              │                          │ device:    │             │
│  │              │                          │  cb_plc    │──────┐      │
│  └──────────────┘                          └─────┬──────┘      │      │
│                                                  │             │      │
│                                           security:      PLC modem   │
│                                                  │       interface   │
│                                           ┌──────┴──────┐     │      │
│                                           │ EvseSecurity │     │      │
│                                           │             │     │      │
│                                           │ V2G certs   │     │      │
│                                           │ CSMS certs  │     │      │
│                                           │ MO certs    │     │      │
│                                           └─────────────┘     │      │
│                                                               │      │
└───────────────────────────────────────────────────────────────┼──────┘
                                                                │
                                                                ▼
                                                    ┌───────────────────┐
                                                    │  PLC Modem (HW)   │
                                                    │  QCA7005 /        │
                                                    │  IS32CG5317       │
                                                    │                   │
                                                    │  ──► CP line ──►  │
                                                    │       to EV       │
                                                    └───────────────────┘
```

### 7.2 EvseV2G Module

Implements the DIN 70121 and ISO 15118-2 SECC (Supply Equipment Communication Controller). Provided by chargebyte GmbH.

**Provides:**
- `charger` → `ISO15118_charger` interface (consumed by EvseManager)
- `extensions` → `iso15118_extensions` interface (consumed by OCPP201 for PnC data exchange)

**Requires:**
- `security` → `evse_security` interface (TLS certs, contract cert validation)

**Configuration:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `device` | string | `eth0` | Network interface for PLC modem. Set `auto` to auto-detect. |
| `supported_DIN70121` | bool | `true` | Enable DIN 70121 |
| `supported_ISO15118_2` | bool | `true` | Enable ISO 15118-2 |
| `tls_security` | enum | `allow` | `prohibit` / `allow` / `force` |
| `verify_contract_cert_chain` | bool | `false` | Local PnC cert verification (usually delegated to OCPP) |
| `auth_timeout_pnc` | int | `55` | Seconds to wait for PnC auth (0 = infinite) |
| `auth_timeout_eim` | int | `300` | Seconds to wait for EIM auth |
| `enable_sdp_server` | bool | `true` | Built-in SECC Discovery Protocol server |
| `tls_key_logging` | bool | `false` | TLS pre-master secret export (Wireshark debug) |

### 7.3 EvseSlac Module

Implements ISO 15118-3 EVSE-side SLAC using `libslac`. Communicates directly with the PLC modem via raw Ethernet socket (`PF_PACKET`).

**Provides:**
- `main` → `slac` interface (consumed by EvseManager)
  - Commands: `enter_bcd`, `leave_bcd`
  - Variables: `dlink_ready`, `dlink_terminate`, `dlink_error`, `dlink_pause`
- `token_provider` → `auth_token_provider` (publishes EV MAC for autocharge)

**Requires:** Nothing (standalone).

**Configuration:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `device` | string | `eth1` | PLC modem interface (must match EvseV2G) |
| `number_of_sounds` | int | `10` | SLAC sounding count (ISO 15118-3 min: 10) |
| `ac_mode_five_percent` | bool | `true` | 5% AC mode retry handling (recommended true even for DC) |
| `set_key_timeout_ms` | int | `1000` | CM_SET_KEY.REQ timeout |
| `link_status_detection` | bool | `false` | Wait for LINK_STATUS MME after match (QCA + Lumissil chips) |
| `reset_instead_of_fail` | bool | `true` | Go to reset (not failed) on abort (handles field EV retries) |
| `publish_mac_on_match_cnf` | bool | `true` | Publish EV MAC on token_provider |

### 7.4 ISO15118_charger Interface — Key Commands and Variables

**Commands called by EvseManager → EvseV2G:**

| Command | When Called | Purpose |
|---------|------------|---------|
| `setup` | Once at startup | Set EVSE ID, SAE J2847 mode |
| `set_charging_parameters` | Per session | AC/DC physical parameters |
| `session_setup` | Per session | Payment options (EIM/PnC) |
| `authorization_response` | Async, after auth | Auth result: Accepted/Rejected |
| `cable_check_finished` | After IMD test | Isolation pass/fail |
| `dlink_ready` | From EvseSlac | PLC link up/down |
| `update_dc_present_values` | Continuous (~100 ms) | Actual V/I from power modules |
| `update_dc_maximum_limits` | On change | EVSE max V/I/P (from power module pool) |
| `update_dc_minimum_limits` | On change | EVSE min V/I |
| `update_isolation_status` | During CableCheck | IMD result enum |
| `stop_charging` | On fault or user stop | Force-stop session |

**Variables published by EvseV2G → EvseManager:**

| Variable | Type | Triggers |
|----------|------|----------|
| `require_auth_eim` | null | EIM auth needed |
| `require_auth_pnc` | ProvidedIdToken | PnC auth needed (includes eMAID + cert chain) |
| `start_cable_check` | null | Begin isolation test |
| `start_pre_charge` | null | Begin voltage ramp |
| `dc_open_contactor` | null | Open DC contactor (session end) |
| `current_demand_started` | null | CurrentDemand loop started |
| `current_demand_finished` | null | CurrentDemand loop ended |
| `dc_ev_target_voltage_current` | DcEvTargetValues | EV's requested V/I |
| `dc_ev_maximum_limits` | DcEvMaximumLimits | EV's max V/I/P limits |
| `dc_ev_status` | DcEvStatus | SoC, error flags, ready state |
| `dc_ev_energy_capacity` | number (Wh) | Battery capacity |
| `dc_ev_energy_request` | number (Wh) | Energy requested |
| `dc_full_soc` / `dc_bulk_soc` | number (%) | Full/bulk SoC targets |
| `dc_charging_complete` | bool | EV signals charge complete |
| `selected_payment_option` | PaymentOption | EIM or Contract |
| `evcc_id` | string | EV MAC address |
| `display_parameters` | DisplayParameters | SoC, capacity for EVSE display |

## 8. YAML Configuration

### 8.1 DC Charger Configuration

```yaml
active_modules:

  # ── SLAC layer (PLC pairing) ──
  slac:
    module: EvseSlac
    config_implementation:
      main:
        device: cb_plc              # PLC modem network interface
        number_of_sounds: 10
        ac_mode_five_percent: true
        reset_instead_of_fail: true
        link_status_detection: true  # For IS32CG5317

  # ── ISO 15118 / DIN 70121 protocol handler ──
  iso15118_charger:
    module: EvseV2G
    config_module:
      device: cb_plc                # Same interface as EvseSlac
      supported_DIN70121: true      # Legacy EV support
      supported_ISO15118_2: true
      tls_security: allow           # TLS optional (force for PnC-only)
      verify_contract_cert_chain: false  # Delegate to OCPP
      auth_timeout_pnc: 55
      auth_timeout_eim: 300
      enable_sdp_server: true
    connections:
      security:
        - module_id: evse_security
          implementation_id: main

  # ── Certificate management ──
  evse_security:
    module: EvseSecurity
    config_module:
      v2g_ca_bundle: ca/v2g/V2G_ROOT_CA.pem
      csms_ca_bundle: ca/csms/CSMS_ROOT_CA.pem
      secc_leaf_cert_directory: client/cso
      secc_leaf_key_directory: client/cso

  # ── Core EVSE Manager ──
  evse_manager:
    module: EvseManager
    config_module:
      connector_id: 1
      charge_mode: DC
      evse_id: DE*PNX*E12345*1
      ac_hlc_enabled: true
      ac_hlc_use_5percent: true
    connections:
      slac:
        - module_id: slac
          implementation_id: main
      hlc:
        - module_id: iso15118_charger
          implementation_id: charger
      bsp:
        - module_id: safety_bsp
          implementation_id: board_support
      powersupply_DC:
        - module_id: powersupply_dc
          implementation_id: main
      imd:
        - module_id: safety_bsp
          implementation_id: imd

  # ── OCPP (connects to ISO 15118 for PnC) ──
  ocpp:
    module: OCPP201
    connections:
      extensions_15118:
        - module_id: iso15118_charger
          implementation_id: extensions
```

### 8.2 Connection Summary

```
EvseManager.slac ──────────────────► slac.main           (SLAC matching)
EvseManager.hlc ───────────────────► iso15118_charger.charger  (V2G protocol)
EvseV2G.security ──────────────────► evse_security.main  (TLS certs)
OCPP201.extensions_15118 ──────────► iso15118_charger.extensions  (PnC data)
```

## 9. Data Flow: ISO 15118 ↔ DCFC Hardware

This traces how ISO 15118 messages translate to hardware actions through the EVerest module graph.

```
ISO 15118 Message         EvseV2G Action              EvseManager Action           Hardware Effect
─────────────────────── ─────────────────────────── ──────────────────────────── ─────────────────────

ChargeParameterDiscovery  publish ev_max_limits       getCapabilities() on        None (informational)
                          receive set_power_caps      powersupply_DC

CableCheckReq             publish start_cable_check   imd.start_self_test()       IMD tests insulation
                                                      bsp.allow_power_on          Contactors close
                                                        (DCCableCheck)            (low voltage)

PreChargeReq              publish start_pre_charge    powersupply_DC.setMode      Single module ramps
                          publish ev_target_V/I         (Precharge)                voltage to target

PowerDelivery(Start)      publish v2g_setup_finished  bsp.allow_power_on          All modules enable
                                                        (FullPowerCharging)

CurrentDemandReq          publish dc_ev_target_V/I    powersupply_DC              Modules deliver
(loop ~60ms)              receive update_dc_present     .setExportVoltageCurrent    requested power
                          receive update_dc_max_lim   Read powermeter             (via CAN #1)

PowerDelivery(Stop)       publish dc_open_contactor   powersupply_DC.setMode(Off) Ramp to 0, open
                                                      bsp.allow_power_on            contactors
                                                        (PowerOff)                (via CAN #2)

SessionStop               publish v2g_setup_finished  connector_lock.unlock()     Unlock motor
                          TCP connection closed        CP → state X1               CP deasserted
```

## 10. Plug & Charge (PnC) Certificate Chain

```
┌───────────────────────────────────────────────────────────────┐
│                    PKI TRUST HIERARCHY                         │
│                                                               │
│  ┌─────────────────────┐     ┌──────────────────────┐        │
│  │   V2G Root CA       │     │   MO Root CA         │        │
│  │   (Hubject / CPO)   │     │  (Mobility Operator) │        │
│  └─────────┬───────────┘     └──────────┬───────────┘        │
│            │                            │                     │
│  ┌─────────▼───────────┐     ┌──────────▼───────────┐        │
│  │   V2G Sub-CA 1      │     │   MO Sub-CA          │        │
│  └─────────┬───────────┘     └──────────┬───────────┘        │
│            │                            │                     │
│  ┌─────────▼───────────┐     ┌──────────▼───────────┐        │
│  │   V2G Sub-CA 2      │     │  Contract Cert       │        │
│  └─────────┬───────────┘     │  (eMAID identity)    │        │
│            │                 │  Lives on EV          │        │
│  ┌─────────▼───────────┐     └──────────────────────┘        │
│  │   SECC Leaf Cert    │                                     │
│  │   (EVSE identity)   │     EV presents Contract Cert       │
│  │   Lives on Phytec SBC      │     to EVSE during TLS handshake    │
│  └─────────────────────┘     EVSE forwards to CSMS via OCPP  │
│                               for validation                  │
│  TLS handshake:                                               │
│    EVSE presents SECC cert → EV validates against V2G Root    │
│    EV presents Contract cert → EVSE/CSMS validates against MO │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

Certificate management is handled by `EvseSecurity`. See [[03 - EVerest OCPP201 Backend Integration#8. EvseSecurity Module: Certificate Management]] for paths and auto-renewal.

## 11. Related Documentation

- [[03 - EVerest OCPP201 Backend Integration]] — OCPP PnC integration, certificate management, auth flow
- [[01 - EVerest Safety Supervisor Integration]] — Contactor sequencing triggered by ISO 15118 phases
- [[02 - EVerest Power Module Driver]] — Power supply providing present V/I values to CurrentDemandRes
- [[04 - Power Module CAN Bus Interface]] — CAN #1 message dictionary for setpoint updates during charging
- [[03 - Safety Supervisor Controller]] — CableCheck/PreCharge contactor sequences
- [[01 - Software Framework]] — EVerest module architecture
- [[02 - Communication Protocols]] — PLC link wiring, CAN bus topology
- [[docs/System/03 - Standards Compliance|03 - Standards Compliance]] — ISO 15118 and DIN 70121 certification
- [[docs/Hardware/07 - CCS Connector and Liquid-Cooled Cable Assembly|07 - CCS Connector and Liquid-Cooled Cable Assembly]] — Physical CP/PP pin connections
- [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]] — EvseManager required connections

## 12. External References

- [EVerest EvseV2G Module Docs](https://everest.github.io/nightly/_generated/modules/EvseV2G.html)
- [EVerest EvseSlac Module Docs](https://everest.github.io/nightly/_included/modules_doc/EvseSlac.html)
- [EVerest ISO15118_charger Interface](https://everest.github.io/contacts-snapshot/generated/interfaces/ISO15118_charger.html)
- [EVerest cbexigen (EXI codec generator)](https://github.com/EVerest/cbexigen)
- [EVerest libslac](https://github.com/EVerest/libslac)
- [EVerest libiso15118](https://github.com/EVerest/libiso15118)
- [Lumissil IS32CG5317](https://www.lumissil.com/applications/automotive/electric-vehicles-charging/IS32CG5317)
- [PHYTEC PEB-X-005 EVSE Expansion Board](https://blog.phytec.com/introducing-the-phytec-evse-expansion-board-peb-x-005)
