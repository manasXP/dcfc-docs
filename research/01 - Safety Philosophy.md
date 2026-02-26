
# Interlocking diagram for better understanding

Below is a **text‑based interlock diagram** tailored to your 150 kW DC charger concept (AC + DC side), showing how hardware safety interlocks sit under CM4/EVerest supervision. This is a conceptual view, not a wiring drawing.[^1][^2][^3][^4][^5]

## High‑level safety chain

```text
          GRID / AC SIDE                         DC / EV SIDE
┌────────────────────────────┐           ┌────────────────────────────┐
│  AC Breaker + RCD          │           │  DC Main Contactor(s)     │
│  (with trip coils)         │           │  DC Precharge Contactor   │
└───────────┬────────────────┘           │  Outlet Contactors (per EV│
            │                            └───────────┬───────────────┘
            ▼                                        │
      HARDWARE TRIP LOOP (no CM4 required)           ▼
```


## Hardware interlock loop (simplified)

This shows the **series safety chain** that must be OK for power contactors to stay closed.[^3][^4][^6][^1]

```text
                             +24 V (Safety Supply)
                                    │
                                    ▼
         ┌────────────────────────────────────────────────┐
         │  SAFETY RELAY / SAFETY PLC (optional, SIL)     │
         │  with forcibly guided contacts                 │
         └───────────────┬────────────────────────────────┘
                         │  Safety loop input
                         ▼
    ┌───────────────────────────────────────────────────────────────┐
    │  EMERGENCY STOP (E-STOP)                                      │
    │  - Normally closed (NC), opens when pressed                   │
    └───────────────┬───────────────────────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────────────────────────────────────┐
    │  DOOR / PANEL INTERLOCK SWITCHES                              │
    │  - NC contacts; open if HV cabinet door is open               │
    └───────────────┬───────────────────────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────────────────────────────────────┐
    │  INSULATION MONITORING DEVICE (IMD) SAFETY CONTACT            │
    │  - Relay closes only if insulation resistance > threshold     │
    │  - If fault → opens, breaking safety loop                     │
    └───────────────┬───────────────────────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────────────────────────────────────┐
    │  RCD / GROUND FAULT RELAY AUX CONTACT                         │
    │  - Closes when no residual fault is detected                     │
    │  - Trip opens contact; loop opens                             │
    └───────────────┬───────────────────────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────────────────────────────────────┐
    │  THERMAL / OVERCURRENT TRIP AUX CONTACTS                      │
    │  - From AC breaker or DC breaker                              │
    │  - Opens on trip                                               │
    └───────────────┬───────────────────────────────────────────────┘
                    │
                    ▼
         ┌────────────────────────────────────────────────┐
         │  SAFETY RELAY OUTPUT CONTACTS                  │
         │  - Drive coils of:                             │
         │     • AC input contactor                       │
         │     • DC main contactor(s)                     │
         │     • Precharge contactor                      │
         └────────────────────────────────────────────────┘
```

If anything in this chain opens, **AC and DC contactors drop**, regardless of what CM5 is doing.[^4][^1][^3]

## How CM4/EVerest connects to this

CM4/EVerest **monitors** but does not own the safety loop.[^2][^5]

```text
              CM4 + EVSE AUX BOARD
┌─────────────────────────────────────────────────────────────┐
│  Inputs (from safety loop, read-only):                      │
│   - E-STOP status (digital input)                           │
│   - Door interlock status                                   │
│   - IMD OK / Fault signal (also via EVerest isolation_monitor)
│   - RCD trip status                                         │
│   - Breaker / contactor AUX feedback                        │
│                                                             │
│  Outputs (non-safety, supervisory):                         │
│   - "Enable request" to safety relay (must be present       │
│      *and* all hardware OK for contactors to energize)      │
│   - CAN commands to DC power bricks (enable/disable, setpoints)
│   - Logic to open session cleanly if any fault detected     │
└─────────────────────────────────────────────────────────────┘
```

Typical behavior:

- CM4 **requests enable** only after all preconditions are OK (EV connected, CP state correct, no faults).[^5][^2]
- If CM4 detects a fault (overcurrent trend, temperature, protocol error), it first **disables bricks over CAN**, then drops its enable request so the safety relay opens the contactors. Hardware still trips if CM4 fails.[^7][^8][^3][^5]


## AC and DC contactor interlock (sequence view)

```text
1. All safety inputs OK → safety loop closed → safety relay ready.
2. CM4 asserts ENABLE to safety relay.
3. Safety relay energizes:
      - AC input contactor.
      - Precharge contactor (via EVSE aux logic).
4. After DC bus charged and verified OK:
      - EVSE aux closes DC main contactor(s) to EV connector.
5. Any fault:
      - Hardware (IMD/RCD/E-STOP/door/thermal) opens loop →
        all contactors drop immediately.
      - Or CM4 removes ENABLE and commands bricks off →
        contactors drop in a controlled way.
```

This **two-layer approach** (hardwired interlock + software supervision) is what is generally recommended for DC fast chargers to meet safety expectations around insulation, earth faults and emergency stop behavior.[^1][^2][^3][^4][^5]

<div align="center">⁂</div>

[^1]: https://www.dold.com/en/company/news/insulation-monitoring-for-dc-charging-stations

[^2]: https://www.einfochips.com/blog/iec-61851-everything-you-need-to-know-about-the-ev-charging-standard/

[^3]: https://library.e.abb.com/public/645395a0a4e041d88204d38ee80874f8/SP_level3_DC_fastChargers_2023.pdf

[^4]: https://www.ti.com/lit/ug/tiduez8c/tiduez8c.pdf?ts=1741033660196

[^5]: https://everest.github.io/nightly/_generated/interfaces/isolation_monitor.html

[^6]: https://www.ledestube.com/understanding-global-standards-for-ev-charging-station-conduits/

[^7]: https://ijcrt.org/papers/IJCRT2506381.pdf

[^8]: https://xray.greyb.com/ev-battery/fault-detection-isolation-evse

