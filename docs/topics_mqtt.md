# MQTT Topics

This is the contract between gateway, consumer and anything downstream. It is deliberately radio-agnostic: if a node were ever migrated to WiFi and published directly, nothing below would change.

## Telemetry

`plants/<plant_id>/telemetry` — one message per node wake cycle. Published by the gateway. Retained, so a restarting consumer immediately knows the last state of every plant.

```json
{
  "ts": "2026-08-09T10:00:00Z",
  "seq": 1043,
  "soil_raw_mv": 1432,
  "soil_temp_c": 21.4,
  "air_temp_c": 24.1,
  "air_rh": 48.2,
  "lux": null,
  "vbat_mv": 2870,
  "rssi": -71,
  "fw": "0.1.0"
}
```

| Field | Notes |
|---|---|
| `ts` | Stamped by the gateway on reception. Nodes have no RTC and no time source. |
| `seq` | Per-node counter in retained RAM. Deduplicates when two gateways hear the same advertisement, and exposes missed samples as gaps. Wraps at 16 bits. |
| `soil_raw_mv` | Uncalibrated on purpose. See [watering logic](watering_logic.md). |
| `soil_temp_c` | From the NTC at the probe tip, not from the SHT40. |
| `lux` | `null` until the VEML7700 is populated (phase 3). |
| `vbat_mv` | Read from VDD via the internal SAADC input. No divider, so no idle drain. |
| `rssi` | Added by the gateway, not the node. Useful for diagnosing coverage. |

`plant_id` is resolved by the gateway from a BLE MAC to `plant_id` map. It is the only per-plant state the gateway holds.

## Derived events

Published by the consumer. These are the interesting outputs — telemetry is just raw material.

`events/plants/<plant_id>/water` — a watering event was detected.

```json
{ "timestamp": "2026-08-09T19:04:00Z", "before_pct": 22.5, "after_pct": 88.1 }
```

`events/plants/<plant_id>/alert` — the plant crossed a threshold, or recovered.

```json
{ "state": "needs_water", "timestamp": "2026-08-09T10:00:00Z", "moisture_pct": 21.4 }
```

`state` is one of `needs_water`, `recovered`, `low_battery`, `sensor_stale`. Phase 3 adds `never_dries`, `poor_absorption` and `probe_drift` — see [watering logic](watering_logic.md).

## Node status

`plants/<plant_id>/status` — reserved. A BLE broadcaster has no last-will, so liveness is derived by the consumer from the absence of telemetry rather than reported by the node. The topic exists so a future mains-powered actuator can use the same namespace.

## Conventions

Carried over from mutual-watchdog:

- Timestamps are RFC 3339 UTC.
- Derived state transitions go under `events/`, raw measurements do not.
- QoS 0 throughout. Samples are cheap and frequent; a lost one is a gap, not an incident.
