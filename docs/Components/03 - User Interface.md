# User Interface

Tags: #dcfc #components #hmi #display #user-interface

Related: [[docs/Hardware/01 - Hardware Components|01 - Hardware Components]] | [[research/04 - Phytec SBC Replacement|04 - Phytec SBC Replacement]]

## 1. Overview

The user interface subsystem provides visual feedback, user interaction, and emergency controls on the front panel of the charger cabinet. The HMI runs a web-based UI served by EVerest on the Phytec SBC.

## 2. Components

| Item | Qty | Description | Manufacturer | Model | Est. Price |
|------|-----|-------------|--------------|-------|------------|
| HD Touchscreen Display | 1 | 14" TFT + capacitive touch, HDMI/USB, sunlight-readable | Waveshare | — | $170 |
| Emergency Stop Button | 1 | Red mushroom-head, hardwired NC, IP65 front panel | — | — | $20 (est.) |
| Status LED Strip | 1 | RGB addressable (WS2812), front panel mounted | — | — | $10 (est.) |

## 3. Subtotal

| Category | Total |
|----------|-------|
| **User Interface** | **$200** |

## 4. Notes

- The touchscreen connects via LVDS + USB to the Phytec SBC and displays the EVerest web UI
- The E-Stop button is hardwired directly into the safety relay chain — no software dependency
- Status LEDs indicate charger state: available (green), charging (blue pulse), error (red), network connected (green)
