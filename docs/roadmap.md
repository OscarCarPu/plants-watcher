# Roadmap

Each phase has an exit criterion. Do not start the next one until it is met — most of the failure modes in this kind of project come from committing to hardware before the sensor has been shown to work.

## Phase 0 — Validate the probe

The riskiest part of the project is not the radio or the firmware, it is whether a cheap capacitive probe produces a usable signal in real soil over weeks. Everything else is well-understood engineering.

- XIAO nRF52840 on a breadboard with a cheap capacitive probe, an NTC and an SHT40
- Throwaway firmware, logging over RTT/`defmt` to CSV — no BLE, no MQTT, no enclosure
- Sample every minute for two weeks in a real pot
- Run two identical probes side by side in the same pot
- Optionally run a commercial sensor (Tuya Zigbee or HHCC Flora, ~18 EUR) alongside as a baseline

**Exit criteria**

- A full dry-down and watering cycle is clearly visible in the data
- Noise is small relative to the dry-to-wet span
- The temperature coefficient is measured, so readings can be compensated
- The two identical probes agree closely enough to trust unit-to-unit consistency

If the probes disagree wildly or the signal is buried in noise, stop and reconsider the sensing approach before spending anything else.

## Phase 1 — v1, one node end to end

Deliberately minimal. One node, one plant, one alert.

- **Node** — Rust `no_std`, `embassy-nrf` + `trouble`. Wake every 10 min, measure soil / NTC / SHT40 / VDD, broadcast three BTHome v2 advertisements, sleep. No connections, no OTA, no remote config, no light sensor.
- **Gateway** — Go. Passive BLE scan, decode BTHome, map MAC to `plant_id`, publish to MQTT.
- **Consumer** — Go in Docker. Subscribe, deduplicate by `(plant_id, seq)`, store in SQLite, apply two-point calibration from `plants.yaml`, send one Telegram alert when a plant goes below threshold and latch it.
- Pure logic in `node/src/lib.rs` and testable Go packages, both runnable on the host. No radio or I/O in either.

**Exit criterion:** seven consecutive days with no missed samples and a correct alert on a real dry-down.

## Phase 2 — Replicate to eight

- Measure real sleep current and confirm the power budget
- Design the PCB: adapt b-parasite's analogue section, swap SHTC3 for SHT40, add the unpopulated VEML7700 footprint and the NTC
- Order 10 assembled boards from JLCPCB
- Enclosure in OpenSCAD, printed in PETG, vented for the SHT40
- Deploy all eight, verify gateway coverage and add an ESP32 bridge if any pot is unreliable

**Exit criterion:** eight nodes reporting for two weeks with no coverage gaps, and measured consumption consistent with multi-year life.

## Phase 3 — Agronomy

All software, no hardware changes. This is where the device stops being an alarm and becomes useful.

- Watering event detection
- MAD from observed field capacity and dry point, replacing fixed thresholds
- Drying rate and days-remaining prediction
- Populate the VEML7700 on new boards; DLI and VPD
- Derived diagnostics: poor absorption, never dries, probe drift, sensor stale

**Exit criterion:** alerts arrive before the plants show stress, without manual threshold tuning.

## Phase 4 — Automatic watering (optional)

Kept separate on purpose. A stuck pump floods a room, which is a different risk class from a missed notification.

- A distinct mains-powered device with WiFi and MQTT — the battery sensor node does not change
- Hardware watchdog, per-dose volume limit, confirmation from the moisture sensor before repeating
- Fails closed

## Risks

| Risk | Mitigation |
|---|---|
| Probe degrades within months | Conformal coating; probe-drift detection so failure is loud, not silent |
| PCB needs a second spin (~115 EUR) | Copy a proven analogue section; validate on breadboard in phase 0 |
| BLE coverage does not reach every pot | ESP32 bridge, ~5 EUR, mains powered |
| Readings meaningless without substrate knowledge | MAD relative to each pot's own observed range, not absolute thresholds |
| Scope creep into apps, ML, automation | The useful product is "tell me when to water, and why" |
