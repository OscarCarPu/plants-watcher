# Hardware

## Decisions

| Function | Part | Why this one |
|---|---|---|
| MCU + radio | **nRF52840** (Ebyte E73-2G4M08S1C module) | ~2 uA sleep, BLE advertising is ~4 orders of magnitude cheaper per sample than WiFi. Runs 1.7-3.6 V, so alkaline cells go straight to the chip with no regulator. `embassy-nrf` is Embassy's reference platform, on stable Rust with `thumbv7em-none-eabihf` — no toolchain fork, no `export-esp.sh`. The module is in JLCPCB's library (`C356849`), so it can be machine-assembled. |
| Soil moisture | **Capacitive electrode on the PCB itself**, driven by the nRF's PWM, rectified into the SAADC | No NE555 daughterboard, no exposed copper, no analogue front-end to invent. Proven by [b-parasite](https://github.com/rbaron/b-parasite). |
| Soil temperature | **NTC 10k** at the probe tip | Capacitive readings drift with temperature and this is what compensates them. Cheap, no protocol, zero idle current. |
| Air temp + humidity | **SHT40** | +/-1.8 %RH, 0.08 uA idle, low long-term drift. In a device meant to run for years, drift matters more than initial accuracy. |
| Light | **VEML7700 footprint, not populated** | Calibrated lux (so real DLI, not a bare phototransistor's arbitrary number). Not needed to decide watering, so it stays unpopulated until phase 3. Costs nothing to leave the pads. |
| Battery sense | **SAADC internal VDD input** | No divider means no resistor pair quietly draining the cells for five years. |
| Power | **2x AAA alkaline, direct** | ~1000 mAh, no LDO, no charger, no protection circuit, no LiPo ageing, replaceable anywhere. Eight rechargeable nodes would be eight chargers and eight things that degrade. |

## Rejected alternatives

**nRF54L15.** The successor, and better on paper (sub-1 uA, Cortex-M33 at 128 MHz). Rejected: immature Rust/Embassy support, small module ecosystem, no USB. The 2 uA to 1 uA difference is irrelevant next to everything else in the budget.

**ESP32-C3 + WiFi.** <a id="why-not-esp32--wifi"></a>The stack from mutual-watchdog, and genuinely tempting to reuse. Rejected on two counts that turned out to be separable:

- *Sleep current.* The ~45 uA measured on a XIAO ESP32-C3 is the board, not the chip — the battery-sense divider and the QSPI flash. A bare module with a low-Iq LDO reaches ~5-8 uA. Fixable without changing radio.
- *Energy per sample.* Associating to WiFi costs ~150 uAh per cycle against ~0.02 uAh for a BLE advertisement. Not fixable. Static IP and a cached BSSID in RTC memory cut it roughly threefold, which lands at about a year on a 1200 mAh LiPo — acceptable, but a LiPo needs an LDO anyway (4.2 V exceeds the C3's 3.6 V limit) and needs recharging.

The deciding factor was neither: it was that a broadcaster has no credentials, no IP and no broker dependency, so eight of them have nothing to invalidate when the network changes.

**Off-the-shelf sensors** (Tuya Zigbee QT-07S ~17 EUR, Xiaomi/HHCC Flora ~18 EUR). Roughly 160 EUR for eight plus a coordinator — the same as building, and working in three days instead of eight weeks. Worth buying one or two anyway as a baseline to validate our readings against. Rejected as the primary path because their probes degrade just the same, the Flora has been discontinued since 2020, and the watering logic still has to be written either way.

**Rechargeable LiPo.** Adds a charger, protection circuit, regulator quiescent current and cell ageing, per node, times eight.

**Bare nRF52840 QFN instead of a module.** Saves ~1.50 EUR per board and costs an RF layout and an unshielded antenna design.

## BOM and cost

Approximate, August 2026, delivered to Spain, VAT excluded. Order of 10 assembled boards (8 deployed, 2 spare).

### Development, one-off

| Item | EUR |
|---|---|
| XIAO nRF52840 (phase 0 breadboarding, later a spare) | 10 |
| Debug probe — ST-Link clone or Pi Pico running `debugprobe` | 4 |
| Cheap capacitive probe for phase 0 validation | 2 |
| BLE USB dongle for the gateway | 8 |
| **Subtotal** | **24** |

### PCBA run, 10 boards

| Item | EUR |
|---|---|
| E73-2G4M08S1C x10 | 40 |
| SHT40 x10 | 25 |
| NTC 10k, passives, battery connector x10 | 5 |
| PCB, 2 layer | 5 |
| PCBA setup + extended part fees | 15 |
| Shipping | 25 |
| **Subtotal (12.50 EUR per assembled board)** | **115** |

### Consumables

| Item | EUR |
|---|---|
| AAA holders + cells x8 | 10 |
| Printed enclosures, filament | 4 |

### Total for 8 pots: ~153 EUR

Against ~160 EUR for eight commercial sensors plus a coordinator. JLCPCB first-PCBA coupons typically bring the run down to ~85 EUR, putting the total nearer 125 EUR.

Three caveats worth keeping in view:

1. The figure assumes the PCB works first time. A second spin is another ~115 EUR and loses the comparison. Mitigate by copying b-parasite's analogue section rather than inventing one, and by validating the probe on a breadboard before ordering.
2. Commercial sensors work in three days; this is 6-10 weeks including fabrication lead time.
3. None of this counts your own time.

If the deployment ends up at three or four pots rather than eight, the fixed costs stop amortising and buying commercial wins clearly.

## Enclosure

Electronics stay above the soil, the probe goes in, joined by the PCB's own narrow neck (b-parasite's layout) rather than a cable. The cable-to-probe joint is where water wicks in, so not having one is the point.

- PETG. PLA degrades with humidity and UV.
- The SHT40 needs a vent. Fully sealing the case means measuring the humidity inside the case, which is useless.
- Conformal coat the probe section even though it has no exposed copper — soldermask is not a long-term moisture barrier, which is exactly how the cheap v1.2 boards fail.
