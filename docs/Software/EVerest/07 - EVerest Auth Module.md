# EVerest Auth Module

Tags: #dcfc #everest #software #auth #rfid #pnc #ocpp #security

Related: [[03 - EVerest OCPP201 Backend Integration]] | [[06 - EVerest EvseManager]] | [[05 - ISO 15118 Vehicle Communication]] | [[05 - EVerest Energy Manager]] | [[01 - Software Framework]] | [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]]

## 1. Overview

The `Auth` module (`modules/EVSE/Auth/`) is the central authorization hub in EVerest. It sits between **token sources** (RFID readers, ISO 15118 Plug and Charge, CSMS remote start) and **token validators** (OCPP backend, local authorization list, authorization cache), routing incoming identity tokens to the appropriate validator and delivering the result to the correct EvseManager.

Auth is the only module that calls `authorize_response` on EvseManager. No other module — not OCPP, not ISO 15118, not RFID — can authorize a session directly. This single-point-of-authorization pattern ensures that all token routing, connector selection, and validation ordering is centralized.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EVerest Module Graph                            │
│                                                                         │
│  Token Sources                    Auth Module                           │
│  ─────────────                    ───────────                           │
│  ┌──────────┐                                                           │
│  │   RFID   │──┐                                                        │
│  │  Reader  │  │  auth_token_     ┌────────────────────────────────┐    │
│  └──────────┘  │  provider        │            Auth                │    │
│                ├─────────────────▶│                                │    │
│  ┌──────────┐  │                  │  requires:                     │    │
│  │ OCPP201  │──┘                  │    token_provider (1-128)      │    │
│  │ auth_    │                     │    token_validator (1-128)     │    │
│  │ provider │                     │    evse_manager (1-128)        │    │
│  └──────────┘                     │                                │    │
│                                   │  provides:                     │    │
│  Token Validators                 │    main → auth                 │    │
│  ────────────────                 │    reservation → reservation   │    │
│  ┌──────────┐                     │                                │    │
│  │ OCPP201  │◀────────────────────│  internal:                     │    │
│  │ auth_    │  auth_token_        │    ConnectorSelector            │    │
│  │ validator│  validator          │    TokenValidator               │    │
│  └──────────┘                     │    ReservationHandler           │    │
│                                   └──────────┬─────────────────────┘    │
│                                              │                          │
│  EVSE Managers                               │ authorize_response()     │
│  ─────────────                               │ withdraw_authorization() │
│  ┌──────────┐                                │ reserve()                │
│  │ EvseMan1 │◀───────────────────────────────┘ cancel_reservation()     │
│  └──────────┘                                                           │
│  ┌──────────┐                                                           │
│  │ EvseMan2 │◀────── (dual-connector config)                            │
│  └──────────┘                                                           │
│                                                                         │
│  OCPP201 (consumes Auth)                                                │
│  ┌──────────┐                                                           │
│  │ OCPP201  │──── auth requirement ──────▶ Auth.main (set timeout,      │
│  │          │                               set master_pass_group_id)    │
│  │          │──── reservation req ───────▶ Auth.reservation              │
│  └──────────┘                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

> [!note] Key principle
> Auth treats all token sources identically. Whether a token comes from an RFID tap, an ISO 15118 contract certificate, or a CSMS `RequestStartTransaction`, it enters Auth through the same `auth_token_provider` interface and follows the same validation and routing logic.

## 2. Module Manifest

```yaml
# modules/EVSE/Auth/manifest.yaml
description: >
  Central authorization hub. Routes tokens from providers (RFID, PnC, CSMS)
  to validators (OCPP, local list), selects the target EVSE, and delivers
  authorization results to EvseManager instances.
config:
  selection_algorithm:
    description: >
      How to select a target connector when a token arrives.
      FindFirst: select first available EVSE.
      PlugEvents: wait for a plug-in event to match token to EVSE.
      UserInput: wait for explicit user selection (HMI).
    type: string
    enum: [FindFirst, PlugEvents, UserInput]
    default: FindFirst
  connection_timeout:
    description: >
      Seconds a pending authorization is valid. If the EV is not plugged in
      within this time, authorization is withdrawn. Synced from CSMS via
      OCPP SetVariables(EVConnectionTimeout).
    type: integer
    default: 30
  master_pass_group_id:
    description: >
      Group ID for tokens that can stop any transaction but cannot start one.
      Used for maintenance cards. Set by CSMS via OCPP SetVariables.
    type: string
    default: ""
  prioritize_authorization_over_stopping_transaction:
    description: >
      When a token is presented and a transaction is active on another
      connector: true = try to start a new session first; false = try
      to stop the existing transaction first.
    type: boolean
    default: true
  ignore_connector_faults:
    description: >
      Allow authorization on faulted connectors. Useful for free-vend
      applications where the operator wants charging to proceed despite
      minor faults.
    type: boolean
    default: false
  plug_in_timeout_enabled:
    description: >
      Whether plug-in events start an authorization timer. When enabled
      with PlugEvents algorithm, the user must plug in within
      connection_timeout after tapping RFID. Useful for multi-EVSE stations.
    type: boolean
    default: false
provides:
  main:
    interface: auth
    description: >
      Authorization control interface. Consumed by OCPP201 to set
      ConnectionTimeout and MasterPassGroupId from CSMS.
  reservation:
    interface: reservation
    description: >
      Reservation management interface. Consumed by OCPP201 for
      ReserveNow and CancelReservation handling.
requires:
  token_provider:
    interface: auth_token_provider
    min_connections: 1
    max_connections: 128
    description: >
      Token sources: RFID readers, OCPP auth_provider (remote start),
      ISO 15118 PnC identity. All sources publish tokens identically.
  token_validator:
    interface: auth_token_validator
    min_connections: 1
    max_connections: 128
    description: >
      Token validators: OCPP auth_validator (CSMS Authorize.req),
      local auth list, authorization cache. Called in order until
      a definitive result is returned.
  evse_manager:
    interface: evse_manager
    min_connections: 1
    max_connections: 128
    description: >
      One connection per EVSE, ordered by EVSE id. Auth calls
      authorize_response, withdraw_authorization, reserve, and
      cancel_reservation on the selected EvseManager.
```

## 3. Interfaces

### 3.1 `auth` Interface (provided as `main`)

Consumed by the OCPP201 module to synchronize authorization settings from the CSMS.

| Command | Arguments | Description |
|---------|-----------|-------------|
| `set_connection_timeout` | `timeout_s: integer` | Update the authorization timeout. Called when CSMS sets `EVConnectionTimeout` via `SetVariables` |
| `set_master_pass_group_id` | `group_id: string` | Update the master pass group ID. Called when CSMS sets `MasterPassGroupId` |

### 3.2 `reservation` Interface (provided as `reservation`)

Consumed by OCPP201 for reservation management per OCPP 2.0.1 `ReserveNow` and `CancelReservation`.

| Command | Arguments | Description |
|---------|-----------|-------------|
| `reserve_now` | `reservation_id`, `id_token`, `evse_id`, `connector_type`, `expiry_date_time` | Reserve an EVSE for a specific token |
| `cancel_reservation` | `reservation_id` | Cancel an existing reservation |

When a reservation is active:
- Auth marks the target EVSE as reserved via `evse_manager.reserve(reservation_id)`
- Only the matching `id_token` (or `group_id_token`) can start a session on that EVSE
- Other tokens are routed to non-reserved EVSEs
- Expired reservations are automatically cleared

### 3.3 `auth_token_provider` Interface (required as `token_provider`)

All token sources implement this interface. Auth subscribes to the `provided_token` variable from each connected provider.

| Variable | Type | Description |
|----------|------|-------------|
| `provided_token` | `ProvidedIdToken` | Published when a new token is presented |

```yaml
ProvidedIdToken:
  id_token:
    value: string              # Token identifier (UID, EMAID, etc.)
    type: string               # Central, eMAID, ISO14443, ISO15693, KeyCode, Local, MacAddress, NoAuthorization
  authorization_type: string   # RFID, PlugAndCharge, BankCard, CSMS (informational)
  connectors: [integer]        # Optional: restrict to specific connector IDs
  prevalidated: boolean        # If true, skip validation (used for NoAuthorization/free-vend)
  request_id: integer          # Optional: correlate with RequestStartTransaction
  parent_id_token: string      # Optional: group token for multi-session matching
```

### 3.4 `auth_token_validator` Interface (required as `token_validator`)

Auth calls each connected validator in order until one returns a definitive result.

| Command | Arguments | Result |
|---------|-----------|--------|
| `validate_token` | `provided_token: ProvidedIdToken`, `evse_id: integer` | `TokenValidationResult` |

```yaml
TokenValidationResult:
  authorization_status: string  # Accepted, Blocked, ConcurrentTx, Expired, Invalid,
                                # NoCredit, NotAllowedTypeEVSE, NotAtThisLocation,
                                # NotAtThisTime, Unknown
  reason: string                # Optional: human-readable reason
  valid_until: string           # ISO 8601 expiry (for cached authorization)
  evse_id: integer              # Optional: override target EVSE
```

## 4. Connector Selection Algorithms

When a token arrives, Auth must decide which EVSE to assign it to. The algorithm is configured via `selection_algorithm`.

### 4.1 FindFirst

```
Token arrives
    │
    ▼
Iterate EVSEs in order (evse_manager connections 0, 1, 2, ...):
    │
    ├── EVSE available (not faulted, not reserved, not in session)?
    │   └── Yes → Select this EVSE → validate token → authorize
    │
    ├── EVSE available but reserved?
    │   └── Does token match reservation? → Yes → Select → validate
    │                                     → No  → Skip
    │
    └── All EVSEs occupied?
        └── If prioritize_authorization_over_stopping_transaction = false:
            Check if token matches an active transaction → stop it
```

Best for: single-connector DCFC stations or stations where connector selection doesn't matter.

### 4.2 PlugEvents

```
Token arrives (RFID tap)
    │
    ▼
Auth stores pending token, starts connection_timeout timer
    │
    ▼
Wait for plug-in event (CP: A→B) from any EvseManager
    │
    ├── Plug-in on EVSE N within timeout:
    │   └── Match token to EVSE N → validate → authorize
    │
    └── Timeout expires:
        └── Withdraw authorization, discard token
```

Best for: dual-connector DCFC stations where user taps RFID then plugs into the desired connector.

### 4.3 UserInput

```
Token arrives
    │
    ▼
Auth publishes selection request to HMI (display_message interface)
    │
    ▼
Wait for user to select connector on touchscreen
    │
    ├── User selects EVSE N:
    │   └── Route token to EVSE N → validate → authorize
    │
    └── Timeout:
        └── Withdraw, discard token
```

Best for: multi-connector stations with a touchscreen HMI.

## 5. Token Validation Flow

### 5.1 RFID Tap → Session Start

```
1. RFID Reader
   User taps RFID card
   RFIDReader publishes ProvidedIdToken:
     { id_token: { value: "04:A2:3B:...", type: "ISO14443" },
       authorization_type: "RFID" }
       │
       ▼
2. Auth Module
   Receives token on token_provider subscription
   Selects target EVSE (FindFirst / PlugEvents / UserInput)
       │
       ▼
3. Token Validation (sequential, first definitive wins)
   Auth calls token_validator[0] (OCPP201 auth_validator):
       │
       ├── Online path:
       │   ├── Check Authorization Cache (LocalPreAuthorize=true)?
       │   │   └── Cache hit → return Accepted immediately (pre-authorize)
       │   │                    (OCPP Authorize.req still sent in background)
       │   │
       │   └── Cache miss → send Authorize.req to CSMS
       │       └── CSMS returns: Accepted / Blocked / Expired / Invalid
       │
       └── Offline path:
           ├── Check Local Auth List (LocalAuthorizeOffline=true)?
           │   └── Found → return Accepted
           │
           └── OfflineTxForUnknownIdEnabled=true?
               └── Yes → return Accepted (allow unknown tokens offline)
               └── No  → return Unknown
       │
       ▼
4. Result Delivery
   If Accepted:
     Auth calls evse_manager.authorize_response(token, Accepted)
     EvseManager transitions Preparing → Authorized
     EvseManager publishes session_event(Authorized)
     OCPP201: TransactionEvent(Started, triggerReason=Authorized)

   If Blocked / Invalid / Expired:
     Auth does NOT call authorize_response
     Auth may publish rejection to HMI (display_message)
     RFID reader LED indicates rejection
```

### 5.2 Plug and Charge (ISO 15118)

```
1. EV plugs in → CP: A→B
   EvseManager: SessionStarted
   SLAC + ISO 15118 TLS handshake

2. EV presents contract certificate (EMAID identity)
   ISO 15118 module extracts EMAID from certificate
   OCPP201 auth_provider publishes ProvidedIdToken:
     { id_token: { value: "DE-ABC-C12345-6", type: "eMAID" },
       authorization_type: "PlugAndCharge" }
       │
       ▼
3. Auth receives token on token_provider subscription
   (Same flow as RFID — Auth doesn't care about the source)
       │
       ▼
4. Token Validation
   Auth calls OCPP201 auth_validator:
     OCPP201 sends Authorize.req with iso15118CertificateHashData
     CSMS validates contract certificate chain (OCSP check)
     Returns: Accepted or CertificateRevoked
       │
       ▼
5. Auth calls evse_manager.authorize_response(token, Accepted)
   EvseManager continues to CableCheck
```

> [!note] PnC configuration
> PnC requires `ISO15118Ctrlr.PnCEnabled=true` and `V2GCertificateInstallationEnabled=true` in the OCPP device model. See [[03 - EVerest OCPP201 Backend Integration#9. ISO 15118 Plug and Charge Integration]] and [[05 - ISO 15118 Vehicle Communication]] for the full certificate chain.

### 5.3 CSMS Remote Start

```
1. CSMS sends RequestStartTransaction.req
   { idToken: { idToken: "04:A2:3B:...", type: "ISO14443" },
     evseId: 1,
     chargingProfile: { ... } }
       │
       ▼
2. OCPP201 module receives via libocpp
   If AuthorizeRemoteStart=false:
     No local validation needed
   auth_provider publishes ProvidedIdToken:
     { id_token: { value: "04:A2:3B:...", type: "ISO14443" },
       authorization_type: "CSMS",
       connectors: [1] }     # CSMS specified evseId=1
       │
       ▼
3. Auth receives token
   CSMS specified connector → skip selection algorithm
   Route directly to EvseManager 1
       │
       ▼
4. Validation (may skip if AuthorizeRemoteStart=false)
   Auth calls evse_manager.authorize_response(token, Accepted)
   EvseManager starts session
   OCPP201: TransactionEvent(Started, triggerReason=RemoteStart)
```

### 5.4 Free-Vend (No Authorization Required)

For installations where charging is free (e.g., fleet depots, employee parking):

```yaml
# Token provider that auto-generates a NoAuthorization token on plug-in
auto_auth:
  module: DummyTokenProvider
  config_module:
    token_type: NoAuthorization
```

Auth receives a `prevalidated: true` token and skips validation entirely.

## 6. Validation Priority and Offline Behavior

### 6.1 Validator Call Order

Auth calls validators in connection order (first `token_validator` connection first). For our DCFC:

```yaml
token_validator:
  - module_id: ocpp          # Validator 0: OCPP201 auth_validator
    implementation_id: auth_validator
```

If multiple validators are wired (e.g., a local validator module for fleet-specific logic), Auth tries each in order until one returns a non-`Unknown` status.

### 6.2 OCPP Validator Decision Tree

```
validate_token(token, evse_id):
    │
    ├── Is CSMS connection active?
    │   │
    │   ├── Yes (online):
    │   │   ├── LocalPreAuthorize=true?
    │   │   │   ├── Token in Authorization Cache?
    │   │   │   │   └── Yes → return Accepted (pre-authorize)
    │   │   │   │         (send Authorize.req in background to update cache)
    │   │   │   └── No → send Authorize.req → wait for response
    │   │   │
    │   │   └── LocalPreAuthorize=false?
    │   │       └── Send Authorize.req → wait for response
    │   │
    │   └── No (offline):
    │       ├── LocalAuthorizeOffline=true?
    │       │   ├── Token in Local Auth List?
    │       │   │   └── Yes → return Accepted
    │       │   └── Token in Authorization Cache?
    │       │       └── Yes → return Accepted
    │       │
    │       └── OfflineTxForUnknownIdEnabled=true?
    │           └── Yes → return Accepted
    │           └── No  → return Unknown (cannot validate)
    │
    └── Return result to Auth
```

### 6.3 Offline Transaction Handling

When authorized offline:
- libocpp generates a UUID-based `transactionId` locally
- All `TransactionEvent.req` messages are queued in SQLite
- Upon CSMS reconnection, queued messages are flushed in order
- If CSMS rejects the offline token retroactively (`Authorize.conf` with `Invalid`), the session may be terminated mid-charge (depends on `StopTxOnInvalidId` device model variable)

### 6.4 Authorization Cache

The OCPP201 module maintains an in-memory/SQLite authorization cache:

| Operation | Trigger |
|-----------|---------|
| Cache write | Successful `Authorize.conf` with `Accepted` |
| Cache invalidate | `Authorize.conf` with `Invalid`, `Blocked`, or `Expired` |
| Cache clear | CSMS sends `ClearCache.req` |
| Cache lifetime | `AuthCacheLifeTime` in `AuthCacheCtrlr.json` (0 = no expiry) |

Pre-authorization from cache saves 200-500 ms of round-trip latency to the CSMS, providing a faster user experience at the charger.

## 7. Reservation Handling

### 7.1 Reserve Flow

```
CSMS sends ReserveNow.req
  { id: 42, idToken: { idToken: "USER-123", type: "Central" },
    evseId: 1, expiryDateTime: "2026-02-27T15:00:00Z" }
    │
    ▼
OCPP201 calls Auth.reservation.reserve_now(...)
    │
    ▼
Auth:
  1. Check EVSE 1 status (available, not faulted, not in session)
  2. If available → call evse_manager[0].reserve(42)
  3. EvseManager marks EVSE as Reserved
  4. EvseManager publishes session_event with Reserved status
  5. OCPP201: StatusNotification(Reserved)
    │
    ▼
Auth stores reservation:
  { id: 42, token: "USER-123", evse: 1, expiry: "...T15:00:00Z" }
```

### 7.2 Reservation Matching

When a token arrives while a reservation is active:

```
Token arrives
    │
    ▼
Does token match any active reservation?
    │
    ├── Yes (token == reservation.id_token or group match):
    │   └── Route to reserved EVSE → validate → authorize
    │       On authorization: reservation consumed (cleared)
    │
    └── No:
        └── Skip reserved EVSEs in connector selection
            Route to non-reserved EVSEs only
```

### 7.3 Reservation Expiry

Auth monitors reservation expiry timestamps. When a reservation expires:
1. `cancel_reservation` called on EvseManager
2. EvseManager publishes `session_event` returning to Available
3. OCPP201: `StatusNotification(Available)`
4. CSMS is NOT explicitly notified (per OCPP spec — CSMS tracks expiry independently)

## 8. Master Pass and Transaction Stop

### 8.1 Master Pass Group

Tokens with `group_id_token == master_pass_group_id` can stop any active transaction but cannot start new ones. Used for maintenance cards:

```
Maintenance card tapped
    │
    ▼
Auth: token.group_id == master_pass_group_id?
    │
    ├── Yes:
    │   └── Iterate active transactions
    │       Call evse_manager.stop_transaction(DeAuthorized)
    │       (Cannot start a new session)
    │
    └── No:
        └── Normal token flow
```

### 8.2 Token-to-Stop Logic

When `prioritize_authorization_over_stopping_transaction = false`:

```
Token arrives, matching active transaction on EVSE 1
    │
    ▼
Auth: prioritize_stopping? Yes (config = false)
    │
    └── Same token as active session?
        ├── Yes → call evse_manager.stop_transaction(Local)
        └── No  → try to start new session on another EVSE
```

When `true` (default): Auth first tries to start a new session on another connector; only if no connector is available does it check if the token matches an active session to stop it.

## 9. Authorization Timeout

The `connection_timeout` parameter (default 30 s, overridable by CSMS) governs how long Auth holds a pending authorization before withdrawing it.

### 9.1 RFID-First Flow (FindFirst)

```
RFID tap → Auth validates → Accepted
    │
    ▼
Auth calls evse_manager.authorize_response(Accepted)
EvseManager enters Preparing state
    │
    ▼
Timer starts: connection_timeout = 30 s
    │
    ├── EV plugs in within 30 s (CP: A→B):
    │   └── Session proceeds normally
    │
    └── 30 s elapses, no plug-in:
        └── Auth calls evse_manager.withdraw_authorization()
            EvseManager: MREC9AuthorizationTimeout
            Session aborted
```

### 9.2 PlugEvents Flow

```
RFID tap → Auth stores pending token
    │
    ▼
Timer starts: connection_timeout = 30 s
    │
    ├── EV plugs in on EVSE 1 within 30 s:
    │   └── Match token to EVSE 1 → validate → authorize
    │
    └── 30 s elapses, no plug-in:
        └── Discard pending token
            No authorization attempted
```

## 10. Integration with Other Modules

### 10.1 OCPP201

OCPP201 has a bidirectional relationship with Auth:

| Direction | Interface | Purpose |
|-----------|-----------|---------|
| OCPP201 → Auth | `auth` (Auth.main) | Set `connection_timeout`, `master_pass_group_id` from CSMS |
| OCPP201 → Auth | `reservation` (Auth.reservation) | Reserve/cancel reservations from CSMS |
| Auth → OCPP201 | `auth_token_provider` (ocpp.auth_provider) | CSMS remote start tokens published to Auth |
| Auth → OCPP201 | `auth_token_validator` (ocpp.auth_validator) | Auth sends tokens to OCPP for CSMS validation |

### 10.2 EvseManager

| Direction | Call | Purpose |
|-----------|------|---------|
| Auth → EvseManager | `authorize_response(token, result)` | Deliver validation result |
| Auth → EvseManager | `withdraw_authorization()` | Revoke on timeout or CSMS deauthorization |
| Auth → EvseManager | `reserve(reservation_id)` | Mark EVSE reserved |
| Auth → EvseManager | `cancel_reservation()` | Clear reservation |
| EvseManager → Auth | `session_event` subscription | Track session lifecycle for cleanup |

Auth subscribes to `session_event` to know when sessions end (for internal bookkeeping, pending token cleanup, and reservation state).

### 10.3 RFID Readers

The RFID reader module provides the `auth_token_provider` interface. Multiple RFID readers can be wired to different `token_provider` slots on Auth (e.g., one reader per connector on a dual-connector DCFC):

```yaml
token_provider:
  - module_id: rfid_connector_1
    implementation_id: main
  - module_id: rfid_connector_2
    implementation_id: main
  - module_id: ocpp
    implementation_id: auth_provider
```

The RFID module itself is hardware-specific (SPI, I2C, or USB to the NFC reader IC). It publishes `ProvidedIdToken` when a card is detected. Common NFC standards supported:

| Standard | Token Type | Example |
|----------|-----------|---------|
| ISO 14443A/B | `ISO14443` | MIFARE Classic, DESFire, NFC Forum Type 2/4 |
| ISO 15693 | `ISO15693` | Vicinity cards |
| FeliCa | `ISO14443` | Sony FeliCa (Japan) |

### 10.4 HMI / Display

Auth can optionally connect to a display module to show:
- "Tap RFID card" prompt (when idle)
- "Authorizing..." (during validation)
- "Authorization accepted" / "Authorization denied" (result)
- "Select connector" (UserInput algorithm)
- Reservation status indicator

This uses the `display_message` interface if connected.

## 11. Multi-Connector Operation

### 11.1 Dual-Connector DCFC

In a dual-connector DCFC, Auth has two `evse_manager` connections:

```yaml
auth:
  module: Auth
  config_module:
    selection_algorithm: PlugEvents    # User taps RFID, then plugs in
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
      - module_id: evse_manager_1       # EVSE 1 (CCS connector 1)
        implementation_id: evse
      - module_id: evse_manager_2       # EVSE 2 (CCS connector 2)
        implementation_id: evse
```

With `PlugEvents`:
1. User taps RFID → Auth holds pending token
2. User plugs into connector 2 → Auth routes token to EvseManager 2
3. Auth validates → authorize EvseManager 2

If the user plugs into a reserved connector that doesn't match their token, Auth ignores that plug event and waits for a plug on an unreserved or matching connector.

### 11.2 Multiple Simultaneous Sessions

Auth tracks active sessions per EVSE. When both connectors have active sessions:
- New RFID tap with `prioritize_authorization_over_stopping_transaction=true`:
  → No available EVSE → reject token (display "Station busy")
- New RFID tap with `prioritize_authorization_over_stopping_transaction=false`:
  → Check if token matches either active session → stop matching session

## 12. YAML Configuration

### 12.1 Single-Connector DCFC (150 kW)

```yaml
active_modules:

  # ── Auth Module ──
  auth:
    module: Auth
    config_module:
      selection_algorithm: FindFirst
      connection_timeout: 30
      prioritize_authorization_over_stopping_transaction: true
      ignore_connector_faults: false
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

  # ── RFID Reader ──
  rfid:
    module: RFIDReader
    config_module:
      device: /dev/spidev0.0           # SPI interface to NFC reader
      reader_type: PN532                # NFC reader IC
      poll_interval_ms: 200

  # ── OCPP201 (provides auth_validator and auth_provider) ──
  ocpp:
    module: OCPP201
    connections:
      auth:
        - module_id: auth
          implementation_id: main
      # ... evse_manager, security, system, etc.

  # ── EvseManager ──
  evse_manager:
    module: EvseManager
    config_module:
      connector_id: 1
      charge_mode: DC
    connections:
      # ... bsp, imd, powersupply_DC, etc.
```

### 12.2 Dual-Connector DCFC with Per-Connector RFID

```yaml
active_modules:

  auth:
    module: Auth
    config_module:
      selection_algorithm: PlugEvents
      connection_timeout: 30
    connections:
      token_provider:
        - module_id: rfid_1             # Reader on connector 1
          implementation_id: main
        - module_id: rfid_2             # Reader on connector 2
          implementation_id: main
        - module_id: ocpp
          implementation_id: auth_provider
      token_validator:
        - module_id: ocpp
          implementation_id: auth_validator
      evse_manager:
        - module_id: evse_manager_1
          implementation_id: evse
        - module_id: evse_manager_2
          implementation_id: evse

  rfid_1:
    module: RFIDReader
    config_module:
      device: /dev/spidev0.0
  rfid_2:
    module: RFIDReader
    config_module:
      device: /dev/spidev1.0

  ocpp:
    module: OCPP201
    connections:
      auth:
        - module_id: auth
          implementation_id: main
      reservation:
        - module_id: auth
          implementation_id: reservation
      evse_manager:
        - module_id: evse_manager_1
          implementation_id: evse
        - module_id: evse_manager_2
          implementation_id: evse
```

### 12.3 Free-Vend Configuration (No RFID, No OCPP Auth)

```yaml
  auth:
    module: Auth
    config_module:
      selection_algorithm: FindFirst
      connection_timeout: 10
    connections:
      token_provider:
        - module_id: auto_auth
          implementation_id: main
      token_validator:
        - module_id: always_accept
          implementation_id: main
      evse_manager:
        - module_id: evse_manager
          implementation_id: evse

  auto_auth:
    module: DummyTokenProvider
    config_module:
      token_type: NoAuthorization

  always_accept:
    module: DummyTokenValidator
```

### 12.4 Connection Map

```
                      Auth Wiring Diagram

  RFID.main ──────────────────────▶ Auth.token_provider[0]
  OCPP201.auth_provider ──────────▶ Auth.token_provider[1]
  OCPP201.auth_validator ◀────────── Auth.token_validator[0]

  Auth ──── authorize_response() ──▶ EvseManager.evse
  Auth ──── withdraw_authorization()▶ EvseManager.evse
  Auth ──── reserve() ─────────────▶ EvseManager.evse

  OCPP201 ── set_connection_timeout ▶ Auth.main
  OCPP201 ── reserve_now ──────────▶ Auth.reservation
```

## 13. Testing

### 13.1 Unit Tests

| Test | Validates |
|------|-----------|
| FindFirst selection | First available EVSE selected |
| FindFirst with reservation | Reserved EVSE skipped, next available selected |
| PlugEvents matching | Correct EVSE selected after plug-in event |
| PlugEvents timeout | Token discarded after `connection_timeout` |
| Validator ordering | First validator's Accepted result used, second not called |
| Validator fallthrough | First returns Unknown, second returns Accepted |
| Master pass stop | Master pass token stops active transaction, cannot start |
| Reservation match | Matching token consumes reservation |
| Reservation expiry | Expired reservation auto-cleared |
| Authorization timeout | `withdraw_authorization` called after timeout |
| Offline: local list | Token in local list → Accepted offline |
| Offline: cache | Token in cache → Accepted offline |
| Offline: unknown | `OfflineTxForUnknownIdEnabled` → Accepted or Unknown |

### 13.2 Integration Test Scenarios

| Scenario | Steps | Expected Result |
|----------|-------|-----------------|
| RFID → charge | Tap card, plug in | Session starts, OCPP TransactionEvent(Started) |
| PnC auto-auth | Plug in with PnC EV | Contract cert validated, session starts |
| Remote start | CSMS sends RequestStartTransaction | EvseManager starts session on specified EVSE |
| Remote stop | CSMS sends RequestStopTransaction | Session terminated orderly |
| Blocked token | Tap blocked card | Auth rejects, no session, HMI shows "Blocked" |
| Expired token | Tap expired card | Auth rejects, CSMS returns Expired |
| Reservation → charge | Reserve EVSE 1, matching card taps, plugs in | Session on EVSE 1, reservation consumed |
| Reservation wrong card | Reserve EVSE 1, different card taps | Routed to EVSE 2 (if available) |
| Reservation expiry | Reserve, wait past expiry | EVSE returns to Available |
| Dual-connector conflict | Both EVSEs active, new RFID tap | "Station busy" or stop matching session |
| CSMS offline + charge | Disconnect CSMS, tap RFID | Local auth list → Accepted, queued events |
| CSMS offline + unknown | Disconnect CSMS, unknown card | Depends on `OfflineTxForUnknownIdEnabled` |
| Master pass stop | Active session, tap maintenance card | Session stopped, card cannot start new |
| Auth timeout | Tap RFID, don't plug in for 30 s | `MREC9AuthorizationTimeout`, withdrawn |
| Free-vend | Auto-auth on plug-in | Session starts immediately, no RFID needed |

## 14. OCPP Device Model Configuration

The following OCPP device model config files affect Auth behavior. These are JSON files in `component_config/standardized/`:

| File | Key Variables | Effect on Auth |
|------|--------------|----------------|
| `AuthCtrlr.json` | `LocalAuthorizeOffline`, `LocalPreAuthorize`, `OfflineTxForUnknownIdEnabled`, `AuthorizeRemoteStart` | Offline behavior, pre-authorization, remote start validation |
| `AuthCacheCtrlr.json` | `Enabled`, `AuthCacheLifeTime`, `AuthCacheStorage` | Authorization cache behavior |
| `LocalAuthListCtrlr.json` | `Enabled`, `Entries`, `BytesPerMessageSendLocalList` | Local authorization list for offline use |
| `ISO15118Ctrlr.json` | `PnCEnabled`, `ContractValidationOffline`, `CentralContractValidationAllowed` | Plug and Charge authorization |

These are managed by the CSMS via `SetVariables` / `GetVariables` and synced to Auth through the OCPP201 module's `auth` interface connection.

## 15. Related Documentation

- [[03 - EVerest OCPP201 Backend Integration]] — Auth validator/provider interfaces, offline behavior, security profiles
- [[06 - EVerest EvseManager]] — `authorize_response` command, session event types, authorization timeout
- [[05 - ISO 15118 Vehicle Communication]] — Plug and Charge certificate chain, EMAID tokens
- [[05 - EVerest Energy Manager]] — Session lifecycle interaction (authorization precedes energy allocation)
- [[01 - Software Framework]] — EVerest module architecture, MQTT IPC
- [[research/05 - EVerest Module Architecture|05 - EVerest Module Architecture]] — Interface contracts, YAML wiring patterns
- [[docs/System/03 - Standards Compliance|03 - Standards Compliance]] — OCPP certification, ISO 15118 compliance
