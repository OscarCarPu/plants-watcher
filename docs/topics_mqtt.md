# MQTT Topics

Publish only. Subscriber: [central-pipeline](https://github.com/OscarCarPu/central-pipeline) on `events/plants/#`.

## Telemetry

Topic: `events/plants/<plant_id>/telemetry` 

```json
{
  "ts": "2026-08-09T10:00:00Z",
  "soil_raw_mv": 1432,
  "soil_temp_c": 21.4,
  "air_temp_c": 24.1,
  "air_rh": 48.2,
  "vbat_mv": 3870
}
```

| Field | Type | Source |
|---|---|---|
| `ts` | string | RFC 3339 UTC, SNTP |
| `soil_raw_mv` | int | Probe |
| `soil_temp_c` | float | NTC |
| `air_temp_c` | float | SHT40 |
| `air_rh` | float | SHT40, % |
| `vbat_mv` | int | Battery |

Raw readings only; cleaning happens downstream (except noise cleaning)
