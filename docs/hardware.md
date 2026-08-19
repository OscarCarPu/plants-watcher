# Hardware

## Sensing chain

Decided, and independent of the radio. Nothing below changes with the phase 2 outcome, which is why it can be validated first.

| Function | Part | Why this one |
|---|---|---|
| Soil moisture | **Capacitive electrode on the PCB itself**, driven by the MCU's PWM and rectified into the ADC | No NE555 daughterboard, no exposed copper, no analogue front-end to invent. Proven by [b-parasite](https://github.com/rbaron/b-parasite). |
| Soil temperature | **NTC 10k** at the probe tip | Capacitive readings drift with temperature and this is what compensates them. Cheap, no protocol, zero idle current. |
| Air temp + humidity | **SHT40** | +/-1.8 %RH, 0.08 uA idle, low long-term drift. In a device meant to run for years, drift matters more than initial accuracy. |
| Light | **VEML7700 footprint, not populated** | Calibrated lux (so real DLI, not a bare phototransistor's arbitrary number). Not needed to decide watering, so it stays unpopulated until phase 3. Costs nothing to leave the pads. |

## MCU and radio — decided at phase 2

Phases 0 and 1 run on a **XIAO ESP32-C3**. That is a decision about speed, not about the final design: it needs no PCB, no bridge, no new toolchain, and its onboard LiPo charging means a battery node without designing a power path first. USB serial/JTAG is built into the chip, so there is no external debug probe either. A working node exists in days, and phase 0's real question — does the probe work in soil — gets answered before anything is committed.

The radio decision comes at phase 2, when eight nodes make the ongoing cost real and phase 1 has produced measured numbers instead of estimates.

| | **ESP32-C3 + WiFi** | **nRF52840 + BLE** |
|---|---|---|
| Energy per cycle | ~90 uAh (board), ~50 uAh (bare module) | ~0.37 uAh |
| Life, hourly sampling | ~18 months on 1200 mAh LiPo; ~2.5 years on a bare module | Shelf life of the cells |
| Batteries | LiPo. 3.0-3.6 V range rules out alkaline: a AAA pair is 1.8 V flat, well under the floor | 2x AAA alkaline direct. 1.7-3.6 V, no regulator |
| Maintenance | Recharge 8 nodes yearly | Replace cells when they eventually die |
| Credentials | WiFi + broker password per node. A router password change means reflashing eight potted devices | None. A broadcaster has no credentials, no IP and no broker dependency |
| Infrastructure | None. The node publishes MQTT itself | A Go BLE bridge in central-pipeline, plus coverage through walls (~10-15 m indoors) |
| Toolchain | Already proven here | `embassy-nrf` on stable Rust, `thumbv7em-none-eabihf`, no toolchain fork |
| Module | ESP32-C3-MINI-1, ~2.50 EUR | Ebyte E73-2G4M08S1C, ~4 EUR, JLCPCB `C356849` |

The honest summary: **WiFi trades battery maintenance for infrastructure you do not have to build.** The four-orders-of-magnitude energy gap is real but it is not the deciding number, because hourly sampling puts even the WiFi node comfortably past a year. The argument that survives the arithmetic is the credentials one — eight potted nodes with WiFi passwords baked in are eight things to reflash when the network changes, and a broadcaster has nothing to invalidate.

What should settle it in phase 2, in order: measured sleep current and measured energy per publish from the phase 1 node; how tolerable a yearly recharge across eight pots actually feels; and whether BLE coverage reaches every pot without a second bridge.

## Rejected alternatives

**nRF54L15.** The successor to the nRF52840, and better on paper (sub-1 uA, Cortex-M33 at 128 MHz). Rejected: immature Rust/Embassy support, small module ecosystem, no USB. The 2 uA to 1 uA difference is irrelevant next to everything else in the budget.

**Off-the-shelf sensors** (Tuya Zigbee QT-07S ~17 EUR, Xiaomi/HHCC Flora ~18 EUR). Roughly 160 EUR for eight plus a coordinator — the same as building, and working in three days instead of weeks. Worth buying one or two anyway as a baseline to validate our readings against. Rejected as the primary path because their probes degrade just the same, the Flora has been discontinued since 2020, and the watering logic still has to be written either way.

**Bare QFN instead of a module.** Saves ~1.50 EUR per board and costs an RF layout and an unshielded antenna design.

**Sampling every 10 minutes.** The interval telemetry instinctively wants, and it changes nothing: soil moisture moves over days, watering still appears as a step change between consecutive samples, and drying-rate fits do not improve with denser points. It costs six times the radio energy for no extra information, and it is the single assumption that made WiFi look impossible.

## BOM and cost

Approximate, August 2026, delivered to Spain, VAT excluded.

### Phase 0 and 1, one-off

| Item | EUR |
|---|---|
| XIAO ESP32-C3 x2 (two probes side by side in phase 0) | 10 |
| Cheap capacitive probe x2 for validation | 4 |
| SHT40 breakout x2 | 10 |
| NTC 10k, passives | 2 |
| LiPo 1200 mAh (phase 1) | 8 |
| **Subtotal** | **34** |

No debug probe and no BLE dongle: the C3 has USB serial/JTAG on chip, and phase 0/1 use no BLE at all. Optionally add a commercial sensor (~18 EUR) as a baseline to validate readings against.

### Phase 2 PCBA run, 10 boards

Depends on the radio decision. Eight deployed, two spare.

| Item | WiFi (ESP32-C3-MINI-1) | BLE (E73-2G4M08S1C) |
|---|---|---|
| Module x10 | 25 | 40 |
| SHT40 x10 | 25 | 25 |
| NTC, passives, connector x10 | 5 | 5 |
| LiPo charge + protection x10 | 10 | — |
| PCB, 2 layer | 5 | 5 |
| PCBA setup + extended part fees | 15 | 15 |
| Shipping | 25 | 25 |
| **Subtotal** | **110** | **115** |
| Cells for 8 nodes | 55 (LiPo 1200 mAh) | 10 (AAA + holders) |
| BLE dongle for the bridge | — | 8 |
| Enclosures, filament | 4 | 4 |
| **Total for 8 pots** | **~200** | **~155** |

Against ~160 EUR for eight commercial sensors plus a coordinator. JLCPCB first-PCBA coupons typically take ~30 EUR off the run.

Three caveats worth keeping in view:

1. The figure assumes the PCB works first time. A second spin is another ~110 EUR and loses the comparison. Mitigate by copying b-parasite's analogue section rather than inventing one, and by validating the probe on a breadboard in phase 0.
2. Commercial sensors work in three days; this is weeks, including fabrication lead time.
3. None of this counts your own time.

If the deployment ends up at three or four pots rather than eight, the fixed costs stop amortising and buying commercial wins clearly.

## Enclosure

Electronics stay above the soil, the probe goes in, joined by the PCB's own narrow neck (b-parasite's layout) rather than a cable. The cable-to-probe joint is where water wicks in, so not having one is the point.

- PETG. PLA degrades with humidity and UV.
- The SHT40 needs a vent. Fully sealing the case means measuring the humidity inside the case, which is useless.
- Conformal coat the probe section even though it has no exposed copper — soldermask is not a long-term moisture barrier, which is exactly how the cheap v1.2 boards fail.
