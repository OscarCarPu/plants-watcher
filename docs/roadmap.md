# Roadmap

Each phase has an exit criterion. Do not start the next one until it is met — most of the failure modes in this kind of project come from committing to hardware before the sensor has been shown to work.

Phases 0 and 1 run on a XIAO ESP32-C3 publishing MQTT directly over WiFi. That is not the long-term radio choice, it is the one that gets real data into central-pipeline in days instead of weeks: no PCB, no bridge to write, and a Rust toolchain already proven in the home lab. The radio is decided in phase 2, from measured numbers — see [hardware](hardware.md#mcu-and-radio--decided-at-phase-2).

## Phase 0 — Validate the probe

The riskiest part of the project is not the radio or the firmware, it is whether a cheap capacitive probe produces a usable signal in real soil over weeks. Everything else is well-understood engineering.

Nothing here is battery powered and nothing is permanent. The board sits on a breadboard on USB, and the firmware is throwaway.

**Milestones**

- [ ] XIAO ESP32-C3 on a breadboard with a cheap capacitive probe, an NTC and an SHT40
- [ ] Throwaway firmware: read all three, publish `events/plants/<id>/telemetry` every minute over WiFi
- [ ] central-pipeline ingests the topic into `raw.mqtt_events` — two weeks of history queryable in SQL rather than in a CSV
- [ ] Two identical probes in the same pot, publishing as two `plant_id`s
- [ ] Run for two weeks in a real pot through at least one full dry-down and watering
- [ ] Characterise the temperature coefficient: same soil, deliberate temperature swing
- [ ] Optionally run a commercial sensor (Tuya Zigbee or HHCC Flora, ~18 EUR) alongside as a baseline

Sampling every minute is far more than the signal needs; it is for characterising noise, and it drops to hourly the moment the node runs on a battery.

**Exit criteria**

- A full dry-down and watering cycle is clearly visible in the data
- Noise is small relative to the dry-to-wet span
- The temperature coefficient is measured, so readings can be compensated
- The two identical probes agree closely enough to trust unit-to-unit consistency

If the probes disagree wildly or the signal is buried in noise, stop and reconsider the sensing approach before spending anything else.

## Phase 1 — v1, one node end to end

Deliberately minimal. One node, one plant, one alert. This is where the node stops being a bench rig and starts living in a pot.

**Milestones**

*Node*

- [ ] Rust firmware on the XIAO ESP32-C3: wake from deep sleep, connect WiFi, publish, sleep
- [ ] Hourly wake cycle; measure soil / NTC / SHT40 / battery
- [ ] `seq` counter in RTC memory, surviving deep sleep
- [ ] Static IP and cached BSSID in RTC memory — association is the entire energy budget, so it is the only thing worth optimising
- [ ] Run from a LiPo, not USB
- [ ] Pure logic in `node/src/lib.rs`, host-testable, with no radio or I/O in it
- [ ] No OTA, no remote config, no light sensor

*central-pipeline*

- [ ] Broker user and ACL entry for the node
- [ ] Subscribe `events/plants/#` and ingest into `raw.mqtt_events`
- [ ] `docs/sources/plants.md` — the staging and mart models, documented as tables
- [ ] Staging model: dedup on (`plant_id`, `seq`), two-point calibration, temperature compensation
- [ ] Mart: moisture percentage per plant over time
- [ ] Telegram alert when a plant crosses its threshold, latched so it fires once and clears on recovery
- [ ] Publish `events/plants/<id>/alert`

**Exit criteria**

- Seven consecutive days with no `seq` gaps
- A correct alert on a real dry-down, and exactly one of it
- Measured battery drain consistent with the [power budget](architecture.md#power-budget)

## Phase 2 — Replicate to eight, and decide the radio

Two things happen here, and the first decides the second. Everything before this point runs on one node with a WiFi radio chosen for speed of iteration; eight nodes is where the ongoing cost of that choice — recharging, credentials, association energy — becomes real and measurable rather than estimated.

**Milestones**

*Decide*

- [ ] Measure real deep-sleep current and real energy per publish on the phase 1 node
- [ ] Extrapolate to eight pots: battery swaps per year, and what a router password change would cost
- [ ] Choose: stay on WiFi with a bare ESP32-C3 module, or move to nRF52840 + BLE
- [ ] Record the decision and the numbers behind it in [hardware](hardware.md)

*If BLE*

- [ ] Write the Go BLE bridge in central-pipeline — see [Bridging](topics_mqtt.md#bridging-phase-2)
- [ ] Verify coverage from one bridge; add a second in a far room if any pot is unreliable

*Either way*

- [ ] Design the PCB: adapt b-parasite's analogue section, SHT40, NTC, unpopulated VEML7700 footprint
- [ ] Order 10 assembled boards from JLCPCB (8 deployed, 2 spare)
- [ ] Enclosure in OpenSCAD, printed in PETG, vented for the SHT40
- [ ] Conformal coat the probe section
- [ ] Deploy all eight, one broker user per node

**Exit criteria**

- Eight nodes reporting for two weeks with no coverage gaps
- Measured consumption consistent with the battery life the chosen radio promised

## Phase 3 — Agronomy

All software, all in central-pipeline. No hardware changes. This is where the device stops being an alarm and becomes useful.

**Milestones**

- [ ] Watering event detection, and `events/plants/<id>/water`
- [ ] Field capacity and dry point observed per pot, replacing the manual calibration pair
- [ ] MAD-based thresholds from each pot's own observed range
- [ ] Drying rate and days-remaining prediction
- [ ] VPD from `air_temp_c` and `air_rh`
- [ ] Populate the VEML7700 on new boards; DLI from integrated `lux`
- [ ] Derived diagnostics: probe drift first, then poor absorption, never dries, sensor stale

**Exit criterion:** alerts arrive before the plants show stress, without manual threshold tuning.

## Phase 4 — Automatic watering (optional)

Kept separate on purpose. A stuck pump floods a room, which is a different risk class from a missed notification.

**Milestones**

- [ ] A distinct mains-powered device with WiFi and MQTT — the battery sensor node does not change
- [ ] Hardware watchdog
- [ ] Per-dose volume limit
- [ ] Confirmation from the moisture sensor before repeating a dose
- [ ] Fails closed on every error path

## Risks

| Risk | Mitigation |
|---|---|
| Probe degrades within months | Conformal coating; probe-drift detection so failure is loud, not silent |
| PCB needs a second spin (~115 EUR) | Copy a proven analogue section; validate on breadboard in phase 0 |
| WiFi nodes need recharging too often | Hourly sampling, not 10-minute; measure in phase 1 and let the phase 2 decision act on it |
| Router password change means reflashing eight potted nodes | The strongest argument for BLE in phase 2; until then, one node, easily reached |
| BLE coverage does not reach every pot, if phase 2 goes that way | Second bridge, ~5 EUR, mains powered |
| Readings meaningless without substrate knowledge | MAD relative to each pot's own observed range, not absolute thresholds |
| Scope creep into apps, ML, automation | The useful product is "tell me when to water, and why" |
