# Roadmap

Each phase has an exit criterion. Do not start the next one until it is met — most failures in this kind of project come from committing to hardware before the sensor has been shown to work.

## Phase 0 — Validate the probe

The riskiest part is whether a cheap capacitive probe produces a usable signal in real soil over weeks. Nothing is battery powered and nothing is permanent: the board sits on a breadboard on USB and the firmware is throwaway.

**Milestones**

- [ ] XIAO ESP32-C3 on a breadboard with a capacitive probe, an NTC and an SHT40 — see [hardware](hardware.md)
- [ ] Throwaway firmware: read all three, publish `events/plants/<id>/telemetry` every minute over WiFi
- [ ] central-pipeline ingests the topic into `raw.mqtt_events`
- [ ] Two identical probes in the same pot, publishing as two `plant_id`s
- [ ] Run for two weeks through at least one full dry-down and watering
- [ ] Characterise the temperature coefficient: same soil, deliberate temperature swing

Sampling every minute is for characterising noise. It drops to hourly once the node runs on a battery.

**Exit criteria**

- A full dry-down and watering cycle is clearly visible in the data
- Noise is small relative to the dry-to-wet span
- The temperature coefficient is measured
- The two probes agree closely enough to trust unit-to-unit consistency

If the probes disagree wildly or the signal is buried in noise, stop and reconsider the sensing approach before spending anything else.

## Phase 1 — One node end to end

One node, one plant, running from a LiPo in a pot.

**Milestones**

- [ ] Rust firmware: wake from deep sleep, measure, connect WiFi, publish, sleep
- [ ] Hourly wake cycle
- [ ] `seq` counter in RTC memory, surviving deep sleep
- [ ] Static IP and cached BSSID in RTC memory
- [ ] Run from a LiPo, not USB
- [ ] Pure logic in `node/src/lib.rs`, host-testable, with no radio or I/O in it
- [ ] Broker user and ACL entry for the node; central-pipeline ingesting `events/plants/#`

**Exit criteria**

- Seven consecutive days with no `seq` gaps
- Measured battery drain consistent with the [power estimate](hardware.md#power)

## Later

Not planned in detail until phase 1 is done.

- **Radio decision** — stay on WiFi or move to BLE, from measured sleep current and energy per publish.
- **PCB and enclosure** — replicate to more pots.
- **Agronomy** — calibration, thresholds and alerts, all in central-pipeline.
- **Automatic watering** — a separate mains-powered device; the sensor node does not change.
