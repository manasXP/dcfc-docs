# 150kW DC Fast Charger Comparison

A comparative analysis of leading 150kW-class DC Fast Chargers against the [[__init|DCFC Design]] specifications.

---

## Products Compared

| # | Manufacturer | Model | Rated Power |
|---|-------------|-------|-------------|
| 1 | **This Design** | DCFC (In-house) | 50–350 kW (modular) |
| 2 | **ABB** | Terra 184 | 180 kW |
| 3 | **Tritium** | PKM150 | 150 kW |
| 4 | **Delta Electronics** | Ultra Fast Charger (UFC) | 150 kW |
| 5 | **Kempower** | C-Station + S-Series | 40–600 kW (modular) |

---

## Detailed Comparison

### 1. Power & Electrical Specifications

| Parameter | This Design | ABB Terra 184 | Tritium PKM150 | Delta UFC 150 | Kempower S-Series |
|-----------|-------------|---------------|----------------|---------------|-------------------|
| **Max Power** | 50–350+ kW (modular 25/50 kW blocks) | 180 kW (90 kW × 2 simultaneous) | 50 / 100 / 150 kW (configurable) | 150 kW | 40–600 kW (modular 40 kW blocks) |
| **AC Input** | 3-phase 400–480 Vac, 50/60 Hz | 480Y/277 Vac ±10%, 60 Hz | 380–480 Vac, 3-phase | 380–415 Vac, 50/60 Hz | 400 Vac, 3-phase |
| **Power Factor** | >0.99 | >0.96 | Not published | 0.99 | Not published |
| **DC Output Range** | 200–1000 Vdc | 150–920 Vdc | 150–920 Vdc (CCS) | 200–1000 Vdc | 50–920 Vdc |
| **Max Current** | 500 A | 200 A (per CCS-1) | 350 A (CCS2) / 200 A (CCS1) | 500 A | 200 A (air) / 500 A (liquid) |
| **Efficiency** | >96% (target) | 95% | ~95% (est.) | >94% | Not published |

### 2. Connectors & Charging

| Parameter | This Design | ABB Terra 184 | Tritium PKM150 | Delta UFC 150 | Kempower S-Series |
|-----------|-------------|---------------|----------------|---------------|-------------------|
| **CCS Combo** | CCS1 / CCS2 | CCS1 (2 ports) | CCS1, CCS2 | CCS2 | CCS1, CCS2 |
| **CHAdeMO** | Optional | CHAdeMO 1.2 | CHAdeMO (125 A) | CHAdeMO (125 A) | Optional |
| **Simultaneous EVs** | Depends on config | 2 (power-shared) | 2 (power-shared) | Up to 4 | Up to 8 satellites |
| **Cable Length** | TBD | 6 m (20 ft) | 6 m | Not published | 5 m or 7 m |
| **Cable Cooling** | Liquid-cooled (high power) | Air-cooled | Liquid-cooled | Air-cooled | Air or Liquid-cooled |

### 3. Cooling Methods (Key Comparison)

| Aspect | This Design | ABB Terra 184 | Tritium PKM150 | Delta UFC 150 | Kempower S-Series |
|--------|-------------|---------------|----------------|---------------|-------------------|
| **Cabinet Cooling** | TBD | Forced air | Liquid-cooled (sealed) | Forced air | Forced air |
| **Cable Cooling** | Liquid-cooled | Air-cooled | Liquid-cooled | Air-cooled | Air or Liquid-cooled |
| **Sealed Enclosure** | TBD | No (vented) | Yes (fully sealed) | No (vented) | Satellite: Yes |
| **Coolant Type** | TBD | N/A | Proprietary | N/A | Glysofor N (~2.5 L) |
| **Dust/Ingress Resistance** | TBD | Moderate (IP54) | Excellent (IP65) | Good (IP55) | Good (IP54) |

> [!tip] Cooling Method Analysis
> - **Air Cooling** (ABB, Delta): Simpler, lower cost, but limits power density. Requires vented enclosures that admit dust and moisture. Suitable for moderate climates and lower current ratings (<200 A per cable).
> - **Liquid Cooling** (Tritium, Kempower optional): Enables sealed enclosures (IP65+), higher current cables (350–500 A), smaller footprints, and better extreme-temperature performance. Higher upfront cost but **up to 37% lower TCO over 10 years** (Tritium data). Reduced maintenance due to no air filters.
> - **Hybrid** (Kempower): Offers flexibility — air-cooled satellites for standard passenger EVs, liquid-cooled satellites for heavy-duty / high-power applications in the same system.

### 4. Environmental & Physical

| Parameter | This Design | ABB Terra 184 | Tritium PKM150 | Delta UFC 150 | Kempower S-Series |
|-----------|-------------|---------------|----------------|---------------|-------------------|
| **IP Rating** | TBD | IP54 | IP65 | IP55 | IP54 (satellite) |
| **IK Rating** | TBD | Not published | Not published | IK08 | IK10 |
| **Temp Range** | 0 to 40°C (min target) | -35 to +55°C | -35 to +50°C | -30 to +55°C | -30 to +50°C |
| **Dimensions** | TBD | 565 × 880 × 1900 mm | Compact (not published) | Not published | Satellite: compact wall/pedestal |
| **Weight** | TBD | ~440–480 kg | Not published | Not published | Satellite: lightweight |

### 5. Communication & Standards

| Parameter | This Design | ABB Terra 184 | Tritium PKM150 | Delta UFC 150 | Kempower S-Series |
|-----------|-------------|---------------|----------------|---------------|-------------------|
| **OCPP** | OCPP (via EVerest) | OCPP 1.6J | OCPP 1.6J / 2.0.1 | OCPP 1.5S / 1.6J | OCPP 1.6J / 2.0.1 |
| **ISO 15118** | Yes (via EVerest) | ISO 15118 Ed.1 (PnC) | ISO 15118 | Not published | ISO 15118 |
| **IEC 61851** | IEC 61851-23/24 | IEC 61851 | IEC 61851 | IEC 61851 | IEC 61851 |
| **Vehicle Comms** | CAN, UART, Ethernet | DIN 70121, ISO 15118 | DIN 70121, ISO 15118 | Not published | DIN 70121, ISO 15118 |
| **Network** | Ethernet (EVerest/MQTT) | GSM/3G/4G, Ethernet | 4G/LTE, Ethernet, WiFi | Ethernet, 4G | 4G/LTE, Ethernet, WiFi |

### 6. Architecture & Modularity

| Parameter | This Design | ABB Terra 184 | Tritium PKM150 | Delta UFC 150 | Kempower S-Series |
|-----------|-------------|---------------|----------------|---------------|-------------------|
| **Architecture** | Modular power blocks (25/50 kW) | Modular (all-in-one cabinet) | Modular (shared DC microgrid) | Modular power blocks | Distributed (power unit + satellites) |
| **Power Sharing** | TBD | Yes (2 ports) | Yes (DC microgrid at 950 Vdc) | Yes (up to 4 EVs) | Yes (dynamic allocation) |
| **Scalability** | Add modules | Fixed cabinet | Add chargers to microgrid | Add modules to cabinet | Add satellites to power unit |
| **Software** | EVerest (open-source) | ABB proprietary | Tritium proprietary | Delta proprietary | Kempower ChargEye (proprietary) |
| **SiC MOSFETs** | Yes | Not confirmed | Not confirmed | Not confirmed | Not confirmed |

---

## Key Differentiators

### ABB Terra 184
- Established brand with massive global install base
- Air-cooled design keeps upfront costs lower
- Dual-port power sharing is straightforward
- Wide operating temperature range (-35 to +55°C)
- Limited to 200 A per cable (air-cooled constraint)

### Tritium PKM150
- **Only fully liquid-cooled, IP65-sealed** DCFC on the market
- DC microgrid architecture (950 Vdc) reduces cabling costs by 50%
- 37% lower total cost of ownership vs air-cooled (10-year)
- Under 2-hour installation time
- Shares 80% components with RTM platform (supply chain efficiency)

### Delta UFC 150
- Supports up to **4 simultaneous EVs** from one cabinet
- Widest DC output range (200–1000 Vdc)
- Highest max current (500 A) among compared products
- Modular design allows starting at lower power and upgrading
- Strong power electronics heritage (Delta is a major power supply manufacturer)

### Kempower S-Series
- **Most flexible architecture** — distributed power unit + satellite model
- Mix air-cooled and liquid-cooled satellites in the same system
- Dynamic power management across up to 8 charging spots
- Liquid-cooled variant supports 400 kW+ and 500 A
- Welding industry heritage (Kemppi Group) for cooling expertise

### This Design (DCFC)
- **Open-source software stack** (EVerest) — no vendor lock-in
- Designed around SiC MOSFETs for >96% target efficiency
- Full 200–1000 Vdc range for 400V and 800V EV compatibility
- MQTT-based microservices architecture for extensibility
- Custom hardware design allows optimization for specific deployment needs

---

## Cooling Method Summary

> [!note] Industry Trend
> The industry is shifting toward **liquid cooling** for chargers above 100 kW. Liquid cooling enables:
> - Higher power density (smaller footprint per kW)
> - Sealed enclosures (IP65+) for harsh environments
> - Thinner, lighter cables at higher currents (350–500 A)
> - Better thermal management in extreme temperatures
>
> However, **air cooling** remains viable for cost-sensitive deployments at moderate power levels (≤200 A per cable), especially in controlled environments.

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| **Air Cooling** | Lower cost, simpler maintenance, fewer failure points | Dust ingress, larger footprint, limited to ~200 A cables, filter replacement needed | Urban/sheltered sites, moderate climates, cost-sensitive deployments |
| **Liquid Cooling** | Sealed enclosure, higher current, compact, extreme temp operation | Higher upfront cost, coolant maintenance, pump failure risk | Highway corridors, harsh environments, high-utilization sites, 800V vehicles |
| **Hybrid** | Flexibility, optimized cost per use case | System complexity, dual maintenance procedures | Mixed fleets (passenger + commercial), scalable deployments |

---

## Design Decision: Air Cooling with Clip-On HVAC Unit

> [!important] Decision
> This design adopts an **air cooling strategy with a clip-on HVAC unit** for cabinet thermal management. The HVAC unit interfaces with the main controller via **CANBus**.

### Rationale

**Why air cooling over liquid cooling:**

1. **Simplicity & Reliability** — Air-cooled systems have fewer failure points. No coolant loops, pumps, reservoirs, or leak risks. This aligns with the ABB Terra 184 and Delta UFC approach, both proven at scale in production deployments.
2. **Lower Manufacturing Cost** — Eliminates the need for coolant plumbing, heat exchangers, and liquid-tight seals throughout the cabinet. Reduces BOM cost and assembly complexity.
3. **Easier Field Serviceability** — No coolant fills, bleeds, or leak inspections. Technicians can service the unit with standard HVAC skills rather than specialized liquid-cooling training.
4. **Modular HVAC Approach** — A clip-on HVAC unit can be replaced as a single field-replaceable unit (FRU) without disassembling the charger internals. Failed cooling can be swapped without taking power electronics offline.
5. **Sealed Cabinet Advantage** — Unlike traditional air-cooled designs that use vented enclosures (IP54), a clip-on HVAC unit enables a **sealed cabinet** with the HVAC handling heat rejection externally. This can achieve **IP55 or higher** while still using air cooling internally — mitigating the dust ingress weakness of conventional air-cooled designs.

**Why clip-on HVAC specifically:**

- The HVAC unit mounts externally on the cabinet, creating a closed-loop internal air circulation. Hot internal air passes through the HVAC heat exchanger; cooled air is returned to the cabinet. External ambient air never enters the electronics enclosure.
- This approach is common in industrial enclosure cooling (e.g., Rittal, Pfannenberg, Seifert) and proven in telecom and power distribution cabinets.
- The clip-on form factor allows different HVAC capacities (e.g., 1 kW, 2 kW, 3 kW cooling) to be selected based on deployment climate without redesigning the cabinet.

### CANBus HVAC Interface

> [!info] Full Specification
> See [[docs/HVAC/04 - HVAC CANBus Interface Specification]] for the complete CAN message dictionary, fault codes, EVerest integration, and control logic.

The HVAC unit communicates with the main controller over **CANBus** (CAN #3, 250 kbps, isolated), enabling:

| Function | Description |
|----------|-------------|
| **Temperature Monitoring** | HVAC reports internal cabinet temperature, condenser temperature, and evaporator temperature to the main controller |
| **Setpoint Control** | Main controller sends target temperature setpoint based on operating mode and power output level |
| **Fault Reporting** | HVAC reports compressor faults, fan failures, refrigerant pressure anomalies, and filter status via CAN fault frames |
| **Power Derating Coordination** | If HVAC cannot maintain target temperature (e.g., extreme ambient), it signals the main controller to derate output power before thermal shutdown |
| **Operating Mode** | Main controller can set HVAC to standby, active cooling, heating (for cold climates), or defrost mode |
| **Diagnostics** | HVAC runtime hours, compressor cycles, and energy consumption available for OCPP-based remote monitoring |

> [!tip] Integration with EVerest
> The HVAC CANBus interface can be exposed as an EVerest module, allowing thermal management to participate in the microservices architecture. The HVAC module publishes temperature telemetry via MQTT, and the energy management module can factor HVAC power consumption into total site load calculations.

### Trade-offs Accepted

| Trade-off | Mitigation |
|-----------|------------|
| Lower max cable current vs liquid-cooled (~200 A air-cooled cable limit) | Target 150 kW operating point; 200 A at 750 Vdc = 150 kW is sufficient for the design target |
| HVAC unit adds external footprint | Clip-on design minimizes protrusion; HVAC is a standard industrial form factor |
| HVAC power consumption (~200–500 W) | Minor compared to 150 kW charger output; included in auxiliary power budget |
| Moving parts (compressor, fans) | Standard HVAC components with well-understood MTBF; field-replaceable as single unit |

---

## Other Recommendations

1. **IP Rating Target**: With sealed cabinet + clip-on HVAC, target IP55 or higher to match/exceed Delta UFC and approach Tritium's IP65.
2. **DC Microgrid**: Evaluate Tritium's 950 Vdc DC microgrid approach for multi-charger sites — reduces cabling costs significantly.
3. **OCPP 2.0.1**: Ensure support alongside 1.6J for future-proofing (Tritium and Kempower already support 2.0.1).
4. **Dynamic Power Management**: Implement power sharing across multiple outlets, similar to Kempower's approach.

---

## Sources

- [ABB Terra 184 Spec Sheet (InCharge)](https://inchargeus.com/wp-content/uploads/2025/02/InCharge_Specsheet_Terra-184_v1.5.pdf)
- [ABB Terra 94/124/184 Data Sheet](https://astsbc.org/wp-content/uploads/2023/07/ABB-Terra-124-184-120kW-Level-3-Charger-SpecsInstall.pdf)
- [Tritium PKM150 Data Sheet](https://tritiumcharging.com/wp-content/uploads/2022/01/PKM150-Data-Sheet.pdf)
- [Tritium PKM150 Brochure (2025)](https://tritiumcharging.com/wp-content/uploads/2025/04/Tritium-PKM150-Brochure.pdf)
- [Delta Ultra Fast Charger](https://emobility.delta-emea.com/en/Ultra-Fast-Charger.htm)
- [Delta EV Charging Solutions](https://www.deltaww.com/en-US/products/EV-Charging)
- [Kempower S-Series Datasheet](https://www.nee.ca/wp-content/uploads/2023/08/Kempower-S-Series-ETL-Datasheet-Rev-2.2-0622.pdf)
- [Kempower Liquid Cooled Satellite](https://kempower.com/solution/liquid-cooled-satellite/)
- [Kempower Station Charger](https://kempower.com/solution/kempower-station-charger/)

#dcfc #research #comparison #cooling #standards
