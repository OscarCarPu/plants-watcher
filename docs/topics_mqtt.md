# MQTT Topics

Everything plants-watcher publishes, and nothing it consumes. Nodes measure and publish; [central-pipeline](https://github.com/OscarCarPu/central-pipeline) owns everything from the broker onward — ingestion, storage, calibration, watering logic and alerts. There is no resident process in this repo.

The contract is deliberately radio-agnostic. A node that publishes MQTT directly over WiFi and a node that broadcasts BLE through a bridge produce the same messages, which is what lets the radio stay an open question until phase 2 without holding up anything downstream.

## Telemetry

`events/plants/<plant_id>/telemetry` — one event per node wake cycle. Retained, so a restarting subscriber immediately knows the last state of every plant.

```json
{
  "ts": "2026-08-09T10:00:00Z",
  "seq": 1043,
  "soil_raw_mv": 1432,
  "soil_temp_c": 21.4,
  "air_temp_c": 24.1,
  "air_rh": 48.2,
  "lux": null,
  "vbat_mv": 3870,
  "rssi": -71,
  "fw": "0.1.0"
}
```

| Field | Type | Description |
| ----- | ---- | ----------- |
| `ts` | `string` | RFC 3339 UTC. Stamped by whatever publishes: the node itself when it has WiFi and SNTP, the bridge on reception when the node is a BLE broadcaster with no clock. |
| `seq` | `int` | Per-node counter, held across deep sleep in RTC memory. Exposes missed samples as gaps and doubles as the dedup key. Wraps at 16 bits. |
| `soil_raw_mv` | `int` | Uncalibrated on purpose. See [watering logic](watering_logic.md). |
| `soil_temp_c` | `float` | From the NTC at the probe tip, not from the SHT40. |
| `air_temp_c` | `float` | SHT40. |
| `air_rh` | `float` | SHT40, percent. |
| `lux` | `float \| null` | `null` until the VEML7700 is populated (phase 3). |
| `vbat_mv` | `int` | Battery voltage. |
| `rssi` | `int \| null` | Link quality, for diagnosing coverage. The node's own WiFi RSSI when it publishes directly; added by the bridge when one exists. |
| `fw` | `string` | Firmware version, so a bad build is identifiable in stored history. |

`plant_id` is compiled into the node when it publishes directly. Behind a bridge it is resolved from a MAC address instead — see [Bridging](#bridging-phase-2).

## Contract

Nodes publish. central-pipeline subscribes to `events/plants/#` and inserts each event into `raw.mqtt_events` untouched, then dedups and calibrates in staging. Nothing in this repo reads MQTT.

Telemetry is published **retained, at QoS 0**. Samples are frequent and individually cheap, so a lost one is a gap rather than an incident, and retention means a subscriber that restarts gets current state for every plant without waiting a full interval.

That has one consequence worth stating plainly, because it is the single hole in central-pipeline's delivery guarantee. Delivery QoS is the minimum of the publish and subscription QoS, so a QoS 0 publish reaches central-pipeline at QoS 0 no matter that it subscribes at QoS 1 with a persistent session. The broker will not queue plants telemetry while the pipeline is down. An outage loses every event except the last retained one per plant — visible afterwards as a `seq` gap, never as silently missing history.

**Duplicates are possible and expected.** A single bridge is enough to produce them under retry, and two bridges hearing the same BLE advertisement produce two MQTT events carrying the same `seq`. Bronze keeps both, since it records what actually arrived; staging deduplicates on (`plant_id`, `seq`), keeping the lowest event id — the same split central-pipeline already applies to its other sources.

Each node needs a broker password and an entry in the [mqtt-broker](https://github.com/OscarCarPu/mqtt-broker) ACL granting write on `events/plants/<plant_id>/telemetry`. A user missing from the ACL connects fine and is then denied every topic, which looks exactly like a broken sensor.

## Bridging (phase 2)

Phases 0 and 1 need no bridge: the node has WiFi and publishes the events above itself.

If phase 2 moves the nodes to BLE, they stop speaking IP entirely and something has to translate. That bridge is a Go program and it belongs in central-pipeline, not here — this repo would then contain firmware and hardware only. Building it is a phase 2 step, conditional on the radio decision:

- Passive BLE scan, so nodes are never woken or asked to respond and stay pure broadcasters. On Linux this needs kernel >= 5.10 and BlueZ >= 5.56.
- Decode BTHome v2 into the field set above.
- Stamp `ts` on reception, since a broadcaster has no clock.
- Map BLE MAC to `plant_id` — the only per-plant state the bridge holds.
- Publish to `events/plants/<plant_id>/telemetry`, retained, QoS 0.

No logic, no storage, no decisions. If it dies you lose samples and nothing is corrupted. Two bridges may run at once for coverage; the dedup rule above is what makes that safe.

## Derived events

Published by central-pipeline from the stored series, not by anything in this repo. They are the interesting outputs — telemetry is just raw material. Documented here until central-pipeline grows a `docs/sources/plants.md`, which is where they belong.

`events/plants/<plant_id>/water` — a watering event was detected.

```json
{ "timestamp": "2026-08-09T19:04:00Z", "before_pct": 22.5, "after_pct": 88.1 }
```

`before_pct` and `after_pct` are calibrated moisture percentages; `timestamp` is RFC 3339 UTC.

`events/plants/<plant_id>/alert` — the plant crossed a threshold, or recovered.

```json
{ "state": "needs_water", "timestamp": "2026-08-09T10:00:00Z", "moisture_pct": 21.4 }
```

`state` is one of `needs_water`, `recovered`, `low_battery`, `sensor_stale`; `timestamp` is RFC 3339 UTC. Phase 3 adds `never_dries`, `poor_absorption` and `probe_drift` — see [watering logic](watering_logic.md).

## Node status

`events/plants/<plant_id>/status` — reserved, nothing publishes it yet.

Liveness is derived downstream from the absence of telemetry rather than reported by the node. A WiFi node could carry a last will, but a node that sleeps through 59 of every 60 minutes is disconnected almost all the time, so a will would fire constantly and mean nothing. The topic exists so a future mains-powered actuator, which really is continuously connected, can use the same namespace.

## Conventions

Shared across the home lab:

- Timestamps are RFC 3339 UTC.
- Everything published for the pipeline lives under `events/`. Topics outside it are plumbing internal to a single project, exchanged between components that ship together, and central-pipeline does not ingest them. plants-watcher has no internal consumer, so it has nothing outside `events/`.
- QoS 0 throughout, with the consequence documented above.
