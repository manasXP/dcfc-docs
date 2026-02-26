# EVerest OCPP 2.0.1 Backend Integration

Tags: #dcfc #everest #ocpp #software #backend #csms #security

Related: [[01 - EVerest Safety Supervisor Integration]] | [[06 - EVerest Power Module Driver]] | [[01 - Software Framework]] | [[02 - Communication Protocols]] | [[docs/System/03 - Standards Compliance|03 - Standards Compliance]]

---

## 1. OCPP201 Module Overview

The `OCPP201` module in EVerest (`modules/EVSE/OCPP201/`) implements OCPP 2.0.1 (and 2.1) communication. It wraps `libocpp` — EVerest's certified C++ OCPP library — and integrates it with the rest of the EVerest charging stack via MQTT-based inter-process communication.

The module is used when an EVerest-based DCFC needs to connect to a Charging Station Management System (CSMS). It handles all protocol logic internally (message serialization, retries, state machines) and delegates system-level operations to other EVerest modules through defined interfaces.

> [!note]
> The module name `OCPP201` is kept for backwards compatibility. It supports both OCPP 2.0.1 and OCPP 2.1. The version negotiated is determined at websocket handshake time via the `Sec-WebSocket-Protocol` header.

---

## 2. OCPP201 Module: manifest.yaml Structure

### 2.1 Module-Level Configuration Parameters

These are set in the EVerest node config file (e.g., `config-sil-ocpp201.yaml`) under the `OCPP201` module entry:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `MessageLogPath` | string | `/tmp/everest_ocpp_logs` | Directory for OCPP message logs |
| `CoreDatabasePath` | string | `/tmp/ocpp201` | SQLite database directory (libocpp persistent state) |
| `DeviceModelDatabasePath` | string | `device_model_storage.db` | SQLite DB for the OCPP device model |
| `EverestDeviceModelDatabasePath` | string | `everest_device_model_storage.db` | SQLite DB for EVerest-owned device model components (EVSE, Connector) |
| `DeviceModelDatabaseMigrationPath` | string | `device_model_migrations` | Path to DB migration files |
| `DeviceModelConfigPath` | string | `component_config` | Directory with device model JSON component configs (must contain `standardized/` and `custom/` subdirectories) |
| `EnableExternalWebsocketControl` | boolean | `false` | Allow external connect/disconnect for debug and OCTT testing |
| `MessageQueueResumeDelay` | integer | `0` | Seconds to delay message queue resume after reconnect (some OCTT tests require CSMS to send first) |
| `CompositeScheduleIntervalS` | integer | `30` | Periodic interval (s) to retrieve and publish composite smart charging schedules; `0` = only on change |
| `RequestCompositeScheduleDurationS` | integer | `600` | Duration (s) for which composite schedules are requested from now |
| `RequestCompositeScheduleUnit` | string | `'A'` | Unit for composite schedule: `'A'` (Amps) for AC, `'W'` (Watts) for DC |
| `DelayOcppStart` | integer | `0` | Milliseconds to delay OCPP startup (allows other EVerest modules to stabilize) |
| `ResetStopDelay` | integer | `0` | Seconds to delay shutdown after reset so CSMS can receive final messages |

### 2.2 Provided Interfaces (what OCPP201 exposes to EVerest)

| Implementation ID | Interface | Purpose |
|---|---|---|
| `auth_validator` | `auth_token_validator` | Validates tokens via CSMS `Authorize.req`, Authorization Cache, or Local Auth List |
| `auth_provider` | `auth_token_provider` | Publishes CSMS-initiated authorization requests (`RequestStartTransaction.req`) into EVerest |
| `data_transfer` | `ocpp_data_transfer` | Allows other modules to initiate `DataTransfer.req` to CSMS |
| `ocpp_generic` | `ocpp` | Generic API to control OCPP service and get/set OCPP data |
| `session_cost` | `session_cost` | Publishes session cost data from CSMS (California Pricing extension) |

### 2.3 Required Interfaces (what OCPP201 depends on)

| Requirement ID | Interface | Connections | Module Typically Used | Purpose |
|---|---|---|---|---|
| `evse_manager` | `evse_manager` | 1–128 | `EvseManager` | One per EVSE; drives `StatusNotification`, `TransactionEvent`, meter values, pause/stop/force-unlock |
| `system` | `system` | 1 | `System` | Firmware updates, log uploads, reset handling, system time sync, boot reason |
| `security` | `evse_security` | 1 | `EvseSecurity` | Certificate management for TLS and ISO 15118 |
| `auth` | `auth` | 1 | `Auth` | Set `ConnectionTimeout` and `MasterPassGroupId` from CSMS |
| `evse_energy_sink` | `external_energy_limits` | 0–129 | `EnergyNode` | Apply smart charging composite schedule limits (one per EVSE + one for EVSE id 0) |
| `data_transfer` | `ocpp_data_transfer` | 0–1 | Custom | Handle incoming `DataTransfer.req` from CSMS with custom logic |
| `display_message` | `display_message` | 0–1 | Display module | `SetDisplayMessage`, `GetDisplayMessage`, `ClearDisplayMessage` from CSMS |
| `reservation` | `reservation` | 0–1 | `Auth` | Reservation management |
| `extensions_15118` | `iso15118_extensions` | 0–128 | ISO15118 module | Share PnC certificate data between ISO 15118 and OCPP modules |

---

## 3. OCPP 2.0.1 Device Model Configuration

Unlike OCPP 1.6 (flat key-value config), OCPP 2.0.1 uses a hierarchical **Device Model** with Components and Variables. In EVerest/libocpp, this is configured via JSON files in `component_config/standardized/` and `component_config/custom/`.

### Key standardized component configs

| Component Config File | Purpose |
|---|---|
| `InternalCtrlr.json` | Charge point identity, CSMS URL, security profile, TLS ciphers |
| `OCPPCommCtrlr.json` | Heartbeat, message timeouts, retry behavior, offline threshold, queue settings |
| `AuthCtrlr.json` | Authorization behavior: offline auth, pre-authorize, remote start auth |
| `AuthCacheCtrlr.json` | Authorization cache settings |
| `LocalAuthListCtrlr.json` | Local authorization list management |
| `ISO15118Ctrlr.json` | Plug and Charge enable/disable, V2G certificate installation |
| `AlignedDataCtrlr.json` | Clock-aligned meter value intervals and measurands |
| `SampledDataCtrlr.json` | Sampled meter value intervals and measurands |
| `SmartChargingCtrlr.json` | Smart charging availability and limits |
| `ChargingStation.json` | Physical station identity |
| `EVSE_1.json` (custom) | EVSE 1 component config |
| `Connector_1_1.json` (custom) | Connector 1 of EVSE 1 |

### Critical InternalCtrlr variables

```json
{
  "ChargePointId": "cp001",
  "NetworkConnectionProfiles": "[{
    \"configurationSlot\": 1,
    \"connectionData\": {
      \"ocppCsmsUrl\": \"wss://csms.example.com:443/cp001\",
      \"ocppVersion\": \"OCPP20\",
      \"ocppTransport\": \"JSON\",
      \"ocppInterface\": \"Wired0\",
      \"securityProfile\": 3,
      \"messageTimeout\": 30
    }
  }]",
  "ChargePointVendor": "MyCompany",
  "ChargePointModel": "DCFC-150kW",
  "ChargePointSerialNumber": "SN-000001",
  "FirmwareVersion": "1.0.0",
  "SupportedOcppVersions": "ocpp2.0.1"
}
```

The `securityProfile` inside `NetworkConnectionProfiles` is the primary security configuration point (see Section 5).

---

## 4. Key OCPP 2.0.1 Message Types for DCFC

### 4.1 Session Lifecycle Messages

| Message | Direction | Trigger |
|---|---|---|
| `BootNotification.req` | CS → CSMS | On startup or reset; includes vendor, model, serial, firmware version |
| `BootNotification.conf` | CSMS → CS | CSMS accepts or rejects; sets heartbeat interval and system time |
| `Heartbeat.req` | CS → CSMS | Periodic keepalive based on `HeartbeatInterval`; CSMS returns current time |
| `StatusNotification.req` | CS → CSMS | Connector status changes (Available, Preparing, Charging, SuspendedEV, Finishing, Reserved, Unavailable, Faulted) |

### 4.2 Authorization Messages

| Message | Direction | Purpose |
|---|---|---|
| `Authorize.req` | CS → CSMS | Validate RFID/EV identity token; includes ISO 15118 certificate hash for PnC |
| `Authorize.conf` | CSMS → CS | Returns `idTokenInfo.status` (Accepted, Blocked, Expired, Invalid, etc.) |
| `RequestStartTransaction.req` | CSMS → CS | CSMS-initiated remote start; can include a `ChargingProfile` |
| `RequestStopTransaction.req` | CSMS → CS | CSMS-initiated remote stop |

### 4.3 Transaction Messages (OCPP 2.0.1 unified model)

OCPP 2.0.1 replaces `StartTransaction`/`StopTransaction` from 1.6 with a single unified message:

**`TransactionEvent.req`** covers the entire transaction lifecycle with a `triggerReason` field:

| `triggerReason` | Meaning |
|---|---|
| `CablePluggedIn` | EV connected physically |
| `EVConnectTimeout` | No EV response after connect |
| `Authorized` | User authorized (RFID tap or PnC) |
| `ChargingStateChanged` | Charging started or stopped |
| `SuspendedEV` | EV not accepting power |
| `SuspendedEVSE` | EVSE unable to deliver |
| `StopAuthorized` | Token used to stop session |
| `EVDeparted` | EV unplugged |
| `Deauthorized` | Auth revoked by CSMS |
| `MeterValueClock` | Periodic clock-aligned meter data |
| `MeterValuePeriodic` | Sampled meter data |
| `RemoteStop` | CSMS-initiated stop |
| `ResetCommand` | Reset during transaction |
| `SignedDataReceived` | Signed meter value received |

The `eventType` field categorizes the event: `Started`, `Updated`, or `Ended`.

In EVerest, the `session_event` variable from `EvseManager` drives this state machine. Libocpp handles the message construction and transmission internally.

### 4.4 Meter Value Messages

| Message | Direction | Purpose |
|---|---|---|
| `MeterValues.req` | CS → CSMS | Standalone meter readings (outside transactions) |
| `TransactionEvent.req` with `meterValue` | CS → CSMS | Meter readings during a transaction |

Measurands are configured via `SampledDataCtrlr` and `AlignedDataCtrlr` device model variables. For DCFC, relevant measurands include:
- `Energy.Active.Import.Register` (kWh delivered)
- `Power.Active.Import` (kW)
- `Current.Import` (A per phase or DC total)
- `Voltage` (V)
- `SoC` (State of Charge from EV via ISO 15118)
- `Temperature`

### 4.5 Device Management Messages

| Message | Direction | Purpose |
|---|---|---|
| `GetVariables.req` | CSMS → CS | Read device model variables (replaces `GetConfiguration` from 1.6) |
| `SetVariables.req` | CSMS → CS | Write device model variables (replaces `ChangeConfiguration` from 1.6) |
| `GetBaseReport.req` | CSMS → CS | Request `FullInventory` or `ConfigurationInventory` report |
| `NotifyReport.req` | CS → CSMS | Report data in response to `GetBaseReport` |
| `GetReport.req` | CSMS → CS | Get filtered custom report |
| `ChangeAvailability.req` | CSMS → CS | Set EVSE or connector operative/inoperative |

### 4.6 Firmware Update Flow

OCPP 2.0.1 firmware updates use signed firmware for security:

1. CSMS sends `UpdateFirmware.req` with download URL and `firmwareSigningCertificate`
2. CS validates the firmware certificate using `EvseSecurity` (`verify_certificate` command)
3. CS sends `FirmwareStatusNotification.req`: `Downloading` → `Downloaded` → `VerifyingSignature` → `Installing` → `Installed`
4. CS sets EVSEs to `Inoperative` before installation begins (EVerest/libocpp handles this automatically)
5. After installation, CS reboots and sends `BootNotification.req` with reason `FirmwareUpdate`

In EVerest, this is implemented via the `system` interface:
- `update_firmware` command: triggered on receipt of `UpdateFirmware.req`
- `allow_firmware_installation` command: called by OCPP201 after EVSEs are inoperative
- `firmware_update_status` variable: consumed to generate `FirmwareStatusNotification.req`

---

## 5. Security Profiles (0–3)

OCPP 2.0.1 defines four security profiles. The profile is set in `NetworkConnectionProfiles.securityProfile` within `InternalCtrlr.json`.

| Profile | Transport | Authentication | Use Case |
|---|---|---|---|
| **0** | ws:// (unsecured) | None | Not allowed by spec; EVerest requires `AllowSecurityLevelZeroConnections: true` override |
| **1** | ws:// (unsecured) | HTTP Basic Auth (ChargePointId:Password) | Development/lab; not for production |
| **2** | wss:// (TLS) | HTTP Basic Auth | CSMS validates CS identity via basic auth; CS validates CSMS server certificate |
| **3** | wss:// (TLS + mTLS) | Client certificate | Mutual TLS; highest security; CS has a client certificate signed by CSMS CA |

### Security profile upgrade path
Profiles can only increase, never decrease. CSMS can send `SetNetworkProfile.req` to upgrade the security profile.

### EvseSecurity configuration per profile

| Security Profile | EvseSecurity Parameters Required |
|---|---|
| 0 or 1 | None (no TLS) |
| 2 | `csms_ca_bundle` (trust anchor for CSMS server cert) |
| 3 | `csms_ca_bundle` + `csms_leaf_cert_directory` + `csms_leaf_key_directory` (client cert/key for mTLS) |

The `UseSslDefaultVerifyPaths` InternalCtrlr flag allows using OS default CA bundles instead of `csms_ca_bundle`.

---

## 6. Smart Charging (ChargingProfile / SetChargingProfile)

Smart charging in OCPP 2.0.1 allows the CSMS to set power/current limits on EVSEs.

### 6.1 Charging Profile Types

| `chargingProfilePurpose` | Scope | Description |
|---|---|---|
| `ChargingStationMaxProfile` | Whole station (evseId=0) | Sets maximum power for entire station |
| `TxDefaultProfile` | EVSE or station | Default limits applied when no transaction-specific profile exists |
| `TxProfile` | Active transaction only | Per-transaction limits |
| `ChargingStationExternalConstraints` | Whole station | External energy management constraints |

### 6.2 Charging Profile Structure

```json
{
  "chargingProfileId": 1,
  "stackLevel": 0,
  "chargingProfilePurpose": "TxDefaultProfile",
  "chargingProfileKind": "Absolute",
  "chargingSchedule": [{
    "id": 1,
    "chargingRateUnit": "W",
    "startSchedule": "2026-02-26T00:00:00Z",
    "chargingSchedulePeriod": [
      { "startPeriod": 0, "limit": 150000 },
      { "startPeriod": 3600, "limit": 100000 }
    ]
  }]
}
```

`chargingProfileKind` options: `Absolute` (fixed start time), `Recurring` (daily/weekly), `Relative` (relative to transaction start).

### 6.3 EVerest Integration

Libocpp calculates the **composite schedule** by stacking all active profiles. The composite schedule is:
- Retrieved periodically every `CompositeScheduleIntervalS` seconds
- Retrieved for a duration of `RequestCompositeScheduleDurationS` seconds ahead
- Published to `evse_energy_sink` modules via `set_external_limits` command
- Unit controlled by `RequestCompositeScheduleUnit` (use `'W'` for DC fast charging)

> [!tip]
> For a DCFC with one EVSE, configure two `evse_energy_sink` connections: one for `evseId=0` (station-level) and one for `evseId=1` (EVSE-level).

---

## 7. Auth Module and Token Validation Flow

### 7.1 Auth Module Configuration

The `Auth` module (`modules/EVSE/Auth/`) is the central authorization hub.

| Config Parameter | Default | Description |
|---|---|---|
| `selection_algorithm` | `FindFirst` | How a connector is chosen for an incoming token: `FindFirst`, `PlugEvents`, `UserInput` |
| `connection_timeout` | (required) | Seconds a pending authorization is valid; synced from CSMS via `SetVariables(EVConnectionTimeout)` |
| `master_pass_group_id` | `""` | Group ID for tokens that can stop any transaction but cannot start one |
| `prioritize_authorization_over_stopping_transaction` | `true` | Whether a new token tries to start vs stop a transaction |
| `ignore_connector_faults` | `false` | Allow authorization on faulted connectors (useful for free-charging applications) |
| `plug_in_timeout_enabled` | `false` | Whether plug-in events start an authorization timer (useful for multi-EVSE stations) |

### 7.2 Auth Module Interfaces

**Provides:**
- `main` (interface: `auth`) — Consumed by `OCPP201` to set `ConnectionTimeout` and `MasterPassGroupId`
- `reservation` (interface: `reservation`) — Consumed by `OCPP201` for reservation management

**Requires:**
- `token_provider` (1–128 connections, interface: `auth_token_provider`) — RFID readers, PnC, OCPP `auth_provider`
- `token_validator` (1–128 connections, interface: `auth_token_validator`) — OCPP `auth_validator`, local list validator
- `evse_manager` (1–128 connections, interface: `evse_manager`) — One per EVSE, ordered by EVSE id

### 7.3 Authorization Flow (RFID Tap to Session Start)

```
1. RFID reader presents token
   → Emits auth_token_provider event (token_provider connection)

2. Auth module receives token
   → Selects target connector (FindFirst or PlugEvents algorithm)

3. Auth module calls each token_validator in order:
   a. auth_validator (OCPP201) is called first
      - Checks Authorization Cache (if LocalPreAuthorize=true and online)
      - Checks Local Authorization List (if LocalAuthorizeOffline=true and offline)
      - Sends Authorize.req to CSMS (if online and not pre-authorized)
      - Returns: Accepted / Blocked / Expired / Invalid / etc.

4. If Accepted:
   → Auth module signals EvseManager to authorize the connector
   → EvseManager enables power relay
   → session_event: Authorized triggers TransactionEvent(Started) via OCPP201

5. OCPP201 sends TransactionEvent.req(eventType=Started, triggerReason=Authorized) to CSMS
```

### 7.4 CSMS-Initiated Remote Start

```
1. CSMS sends RequestStartTransaction.req
   → OCPP201 receives via libocpp
   → auth_provider publishes token into EVerest (if AuthorizeRemoteStart=false, no local validation)
   → Auth module routes to available connector
   → EvseManager enables session
```

### 7.5 Authorization Priority (offline fallback)

When the CSMS is unreachable, authorization follows this precedence:
1. Local Authorization List (if `LocalAuthListCtrlr.Enabled=true`)
2. Authorization Cache (if `LocalPreAuthorize=true`)
3. `OfflineTxForUnknownIdEnabled=true` — allow unknown tokens offline

---

## 8. EvseSecurity Module: Certificate Management

The `EvseSecurity` module (`modules/EVSE/EvseSecurity/`) implements the `evse_security` interface, wrapping `libevse-security`. It is the single point of truth for all certificates and private keys.

### 8.1 Configuration Parameters

| Parameter | Default | Description |
|---|---|---|
| `csms_ca_bundle` | `ca/csms/CSMS_ROOT_CA.pem` | Trust anchor for CSMS TLS server certificate (Profile 2 and 3) |
| `mf_ca_bundle` | `ca/mf/MF_ROOT_CA.pem` | Trust anchor for manufacturer firmware signing certificates |
| `mo_ca_bundle` | `ca/mo/MO_ROOT_CA.pem` | Trust anchor for Mobility Operator contract certificates (PnC) |
| `v2g_ca_bundle` | `ca/v2g/V2G_ROOT_CA.pem` | Trust anchor for ISO 15118 TLS between CS and EV |
| `csms_leaf_cert_directory` | `client/csms` | Client certificate for mTLS (Security Profile 3) |
| `csms_leaf_key_directory` | `client/csms` | Private key for mTLS client certificate |
| `secc_leaf_cert_directory` | `client/cso` | SECC server certificate for ISO 15118 TLS |
| `secc_leaf_key_directory` | `client/cso` | SECC private key |
| `private_key_password` | `""` | Password for encrypted private keys |

All paths relative to `<everest_prefix>/etc/everest/certs/` unless absolute.

### 8.2 Certificate Domains

| Domain | CA Bundle | Purpose |
|---|---|---|
| V2G | `v2g_ca_bundle` | ISO 15118 TLS between CS (SECC) and EV (PEV) |
| CSMS | `csms_ca_bundle` | TLS between CS and OCPP CSMS backend |
| MF (Manufacturer) | `mf_ca_bundle` | Verify signed firmware updates |
| MO (Mobility Operator) | `mo_ca_bundle` | Verify EV contract certificates for Plug and Charge |

### 8.3 Automatic Certificate Renewal

60 seconds after a successful `BootNotification.req` response:
- CS checks if CSMS leaf certificate (for Security Profile 3) is missing or expired
  - If so: sends `SignCertificate.req` with a CSR → CSMS responds with `CertificateSigned.req`
- CS checks if SECC leaf certificate is missing or expired (only if `ISO15118Ctrlr.V2GCertificateInstallationEnabled=true`)
  - If so: sends `SignCertificate.req` with V2G CSR

Expiry checks repeat every 12 hours (`ClientCertificateExpireCheckIntervalSeconds`, `V2GCertificateExpireCheckIntervalSeconds`).

OCSP revocation status for V2G sub-CA certificates is updated every 7 days (`OcspRequestInterval`, default 604800 seconds).

---

## 9. ISO 15118 Plug and Charge Integration

### 9.1 Enable PnC in EVerest

In `ISO15118Ctrlr.json` component config:

```json
{
  "PnCEnabled": { "value": true },
  "V2GCertificateInstallationEnabled": { "value": true },
  "ContractCertificateInstallationEnabled": { "value": true },
  "CentralContractValidationAllowed": { "value": true },
  "ContractValidationOffline": { "value": false }
}
```

### 9.2 PnC Flow

```
1. EV connects and initiates ISO 15118 TLS session
   → SECC (CS) presents SECC leaf certificate signed by V2G CA
   → EV validates SECC certificate against V2G root

2. EV presents contract certificate (EMAID-based identity)
   → ISO 15118 module publishes iso15118_certificate_request event
   → OCPP201 receives via extensions_15118 interface
   → OCPP201 sends DataTransfer.req(Get15118EVCertificateRequest) to CSMS if ContractCertificateInstallationEnabled

3. For Authorize.req (PnC path):
   → EvseSecurity.get_mo_ocsp_request_data retrieves OCSP hash of contract cert chain
   → Authorize.req includes iso15118CertificateHashData
   → CSMS validates and returns Accepted or CertificateRevoked

4. If CentralContractValidationAllowed=true:
   → CS can forward contract certificate to CSMS even if it cannot validate locally
```

### 9.3 OCPP201 to ISO15118 Module Connection

The `extensions_15118` requirement (0–128 connections) must be connected to the ISO 15118 module(s) in the EVerest config. One connection per ISO 15118 instance (one per EVSE on a multi-EVSE charger).

---

## 10. Offline Behavior and Message Queue

### 10.1 Offline Detection

The `OCPPCommCtrlr.OfflineThreshold` variable (default: 60 seconds) defines when the charging station is considered offline. After this duration, all connectors send `StatusNotification.req` upon reconnection.

### 10.2 Message Queue During Disconnection

| Variable | Location | Description |
|---|---|---|
| `QueueAllMessages` | `OCPPCommCtrlr.json` | If `true`, all messages are queued during offline periods |
| `MessageTypesDiscardForQueueing` | `OCPPCommCtrlr.json` | Comma-separated list of message types NOT queued even if `QueueAllMessages=true` |
| `MessageQueueSizeThreshold` | `InternalCtrlr.json` | Max in-memory queue size; messages dropped per spec if exceeded |
| `MessageQueueResumeDelay` | OCPP201 manifest | Seconds to delay queue flush after reconnect (for OCTT compliance) |

Transaction-related messages (`TransactionEvent.req`) are always queued when offline. The `MessageAttempts` and `MessageAttemptInterval` variables in `OCPPCommCtrlr.json` control retry behavior:
- `MessageAttempts(TransactionEvent)`: default 5 retries
- `MessageAttemptInterval(TransactionEvent)`: default 10 seconds between retries

### 10.3 Reconnection Behavior

Reconnect uses exponential backoff:
- `RetryBackOffWaitMinimum`: minimum initial wait (default: 1s)
- `RetryBackOffRepeatTimes`: number of doublings (default: 2)
- `RetryBackOffRandomRange`: maximum random jitter added (default: 2s)

After `NetworkProfileConnectionAttempts` (default: 3) failures, the station tries the next `NetworkConfigurationPriority` profile.

### 10.4 Offline Transaction Handling

Controlled via `AuthCtrlr` device model variables:

| Variable | Description |
|---|---|
| `LocalAuthorizeOffline` | Start transactions for locally-authorized IDs while offline |
| `LocalPreAuthorize` | Use Authorization Cache to authorize without waiting for CSMS response (while online) |
| `OfflineTxForUnknownIdEnabled` | Allow tokens not in Local Auth List or Cache to start transactions offline |

When offline, libocpp generates a UUID-based `transactionId` locally. All `TransactionEvent.req` messages are queued and sent upon reconnection. The `ResumeTransactionsOnBoot` flag in `InternalCtrlr` allows resuming transactions that were active before a power cycle.

---

## 11. OCPP 2.0.1 vs OCPP 1.6: Key Implementation Differences

| Aspect | OCPP 1.6 | OCPP 2.0.1 |
|---|---|---|
| **Transaction model** | Separate `StartTransaction.req`, `StopTransaction.req`, `MeterValues.req` | Unified `TransactionEvent.req` with `eventType` (Started/Updated/Ended) and `triggerReason` |
| **Charging hierarchy** | Station → Connectors | Station → EVSEs → Connectors (3-tier) |
| **Configuration** | Flat key-value (`GetConfiguration`/`ChangeConfiguration`) | Hierarchical Device Model (`GetVariables`/`SetVariables`/`GetBaseReport`) |
| **Security** | Basic Auth only (no TLS in spec core) | Security Profiles 1–3 with TLS and mTLS built-in |
| **ISO 15118 PnC** | Not supported | Native support via `Authorize.req` with certificate hash and `DataTransfer` for cert installation |
| **Smart charging** | `SetChargingProfile`, limited profile purposes | Same concept but with `ChargingStationExternalConstraints`, more profile kinds |
| **Offline authorization** | RFID whitelist, limited | Local Auth List + Auth Cache with complex IdToken types and defined buffer/upload rules |
| **Error reporting** | `StatusNotification.req` with `errorCode` | `NotifyEvent.req` using MRECS error classification; `StatusNotification` is status-only |
| **Firmware updates** | `UpdateFirmware.req`, no signature | `UpdateFirmware.req` with `firmwareSigningCertificate`; CS must validate before installing |
| **Backward compatibility** | — | Not backward compatible; separate WebSocket subprotocol (`ocpp2.0.1` vs `ocpp1.6`) |
| **Protocol complexity** | Lower | Significantly higher; Device Model and PnC add substantial implementation complexity |

> [!warning]
> OCPP 1.6 and 2.0.1 are mutually exclusive on a single WebSocket connection. A DCFC must choose one version per CSMS connection. EVerest supports both via separate modules (`OCPP` for 1.6, `OCPP201` for 2.0.1/2.1).

---

## 12. EVerest Configuration Example (OCPP201 Module Section)

In `config-sil-ocpp201.yaml`:

```yaml
OCPP201:
  module: OCPP201
  config_module:
    MessageLogPath: /var/log/everest/ocpp_logs
    CoreDatabasePath: /var/lib/everest/ocpp201
    DeviceModelDatabasePath: /var/lib/everest/device_model.db
    DeviceModelConfigPath: /etc/everest/ocpp201/component_config
    CompositeScheduleIntervalS: 30
    RequestCompositeScheduleDurationS: 600
    RequestCompositeScheduleUnit: 'W'
  connections:
    evse_manager:
      - module_id: EvseManager1
        implementation_id: evse
    system:
      - module_id: System
        implementation_id: main
    security:
      - module_id: EvseSecurity
        implementation_id: main
    auth:
      - module_id: Auth
        implementation_id: main
    evse_energy_sink:
      - module_id: EnergyManager
        implementation_id: energy_sink
        mapping:
          evse: 0          # station-level
      - module_id: EnergyManager
        implementation_id: energy_sink
        mapping:
          evse: 1          # EVSE 1
    extensions_15118:
      - module_id: ISO15118
        implementation_id: main
```

---

## 13. DCFC System Integration Context

### 13.1 Where OCPP Fits in the DCFC Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          CM5 (Linux + EVerest)                            │
│                                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────────┐   │
│  │  OCPP201   │  │    Auth    │  │  EvsSec    │  │    System        │   │
│  │            │  │            │  │            │  │  (FW update,     │   │
│  │ auth_valid.│◀─│ token_val. │  │ certs/keys │  │   log upload,    │   │
│  │ auth_prov. │─▶│ token_prov │  │            │  │   reboot)        │   │
│  │ session_   │  │            │  │            │  │                  │   │
│  │ cost       │  │            │  │            │  │                  │   │
│  └─────┬──────┘  └─────┬──────┘  └────────────┘  └──────────────────┘   │
│        │               │                                                 │
│        │ evse_manager   │ evse_manager                                    │
│        ▼               ▼                                                 │
│  ┌──────────────────────────────────────────┐   ┌────────────────────┐   │
│  │             EvseManager                   │   │   EnergyManager    │   │
│  │                                          │   │                    │   │
│  │  session_event → TransactionEvent        │   │ evse_energy_sink ◀─│───│── OCPP smart
│  │  powermeter    → MeterValues             │   │ (composite sched.) │   │   charging
│  │  hw_capabilities → StatusNotification    │   │                    │   │   profiles
│  └──────┬──────────────────────┬────────────┘   └────────────────────┘   │
│         │                      │                                         │
│         ▼                      ▼                                         │
│  SafetySupervisorBSP    PowerModuleDriver                                │
│  (CAN #2)               (CAN #1)                                         │
│                                                                          │
└──────────┬───────────────────────┬──────────────────┬────────────────────┘
           │                       │                  │
      CAN #2                  CAN #1            ETH / 4G / 5G
      Safety                  Power               │
      Supervisor              Modules              │
                                                   ▼
                                          ┌──────────────────┐
                                          │   CSMS Backend   │
                                          │  (WebSocket/TLS) │
                                          └──────────────────┘
```

### 13.2 OCPP Events from DCFC Hardware

The OCPP module does not interact with hardware directly. All hardware events reach it through EvseManager:

| Hardware Event | Source | EvseManager Variable | OCPP Message |
|---------------|--------|---------------------|--------------|
| EV plugs in | Safety supervisor → BSP (CP state B) | `session_event: Authorized` | `TransactionEvent(Started)` |
| Charging begins | Power modules ramp up | `session_event: ChargingStarted` | `TransactionEvent(Updated, ChargingStateChanged)` |
| Meter update | Power module driver | `powermeter` | `TransactionEvent(Updated, MeterValuePeriodic)` |
| Module fault | Power module CAN #1 | `session_event: Error` | `StatusNotification(Faulted)` |
| Safety fault | Safety supervisor CAN #2 | `session_event: Error` | `StatusNotification(Faulted)` + `NotifyEvent` |
| EV departs | Safety supervisor → BSP (CP state A) | `session_event: SessionFinished` | `TransactionEvent(Ended, EVDeparted)` |
| Thermal derating | HVAC module / power modules | Reduced `capabilities` | CSMS sees reduced `MeterValues` |
| E-STOP | Hardware interlock chain | `session_event: Error` | `StatusNotification(Faulted)` |

### 13.3 Smart Charging Integration with Power Modules

When the CSMS sends a `SetChargingProfile` limiting station power:

```
1. CSMS → SetChargingProfile.req (e.g., limit to 100 kW for 1 hour)
2. libocpp calculates composite schedule
3. OCPP201 publishes limit to evse_energy_sink
4. EnergyManager applies enforce_limits on EvseManager
5. EvseManager calls powersupply_DC.setExportVoltageCurrent() with reduced current
6. PowerModuleDriver recalculates per-module distribution
   (may shed modules to standby for efficiency)
7. Actual output drops to 100 kW
8. TransactionEvent(Updated) with new MeterValues confirms reduced power
```

### 13.4 Network Configuration for the DCFC

| Interface | Connection | Purpose |
|-----------|------------|---------|
| ETH0 (CM5) | Wired Ethernet via PoE switch | Primary CSMS connection |
| 4G/5G modem | SFP module on PoE switch | Backup CSMS connection (failover) |
| DNS/NTP | Via ETH0 or 4G | Time sync (critical for TLS and billing) |

Configure two `NetworkConnectionProfiles` in `InternalCtrlr.json`:

```json
"NetworkConnectionProfiles": "[
  {
    \"configurationSlot\": 1,
    \"connectionData\": {
      \"ocppCsmsUrl\": \"wss://csms.operator.com/cp/DCFC-001\",
      \"ocppVersion\": \"OCPP20\",
      \"ocppTransport\": \"JSON\",
      \"ocppInterface\": \"Wired0\",
      \"securityProfile\": 3,
      \"messageTimeout\": 30
    }
  },
  {
    \"configurationSlot\": 2,
    \"connectionData\": {
      \"ocppCsmsUrl\": \"wss://csms.operator.com/cp/DCFC-001\",
      \"ocppVersion\": \"OCPP20\",
      \"ocppTransport\": \"JSON\",
      \"ocppInterface\": \"Wireless0\",
      \"securityProfile\": 3,
      \"messageTimeout\": 60
    }
  }
]"
```

`NetworkConfigurationPriority` should be `[1, 2]` — try wired first, failover to cellular.

---

## 14. Related Documentation

- [[01 - EVerest Safety Supervisor Integration]] — Safety BSP module that publishes session events consumed by OCPP
- [[06 - EVerest Power Module Driver]] — Power supply driver providing meter values and power control
- [[01 - Software Framework]] — EVerest framework architecture and module system
- [[02 - Communication Protocols]] — Network wiring, OCPP architecture diagrams, CAN bus topology
- [[docs/System/03 - Standards Compliance|03 - Standards Compliance]] — OCPP standard versions and certification requirements
- [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]] — EVerest interface specifications

## 15. External References

- [EVerest OCPP201 Module Docs](https://everest.github.io/nightly/_included/modules_doc/OCPP201.html)
- [EVerest libocpp GitHub](https://github.com/EVerest/libocpp)
- [libocpp OCPP 2.x Status](https://github.com/EVerest/libocpp/blob/main/doc/v2/ocpp_2x_status.md)
- [libocpp Smart Charging In Depth](https://github.com/EVerest/libocpp/blob/main/doc/v2/ocpp_201_smart_charging_in_depth.md)
- [libocpp Device Model Initialization](https://github.com/EVerest/libocpp/blob/main/doc/v2/ocpp_201_device_model_initialization.md)
- [OCA — What is New in OCPP 2.0.1](https://openchargealliance.org/wp-content/uploads/2024/01/new_in_ocpp_201-v10.pdf)
