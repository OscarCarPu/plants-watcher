# MQTT Topics

Everything plants-watcher publishes, and nothing it consumes. Nodes measure and publish; [central-pipeline](https://github.com/OscarCarPu/central-pipeline) subscribes to `events/plants/#` and owns everything from the broker onward.

## Telemetry

`events/plants/<plant_id>/telemetry` — one event per node wake cycle. Published retained, QoS 0.

```json
{
  "ts": "2026-08-09T10:00:00Z",
  "seq": 1043,
  "soil_raw_mv": 1432,
  "soil_temp_c": 21.4,
  "air_temp_c": 24.1,
  "air_rh": 48.2,
  "vbat_mv": 3870,
  "rssi": -71,
  "fw": "0.1.0"
}
```

| Field | Type | Description |
| ----- | ---- | ----------- |
| `ts` | `string` | RFC 3339 UTC, stamped by the node via SNTP. |
| `seq` | `int` | Per-node counter held across deep sleep in RTC memory. Missed samples show up as gaps. Wraps at 16 bits. |
| `soil_raw_mv` | `int` | Uncalibrated on purpose. Calibration happens downstream. |
| `soil_temp_c` | `float` | From the NTC at the probe tip, not from the SHT40. |
| `air_temp_c` | `float` | SHT40. |
| `air_rh` | `float` | SHT40, percent. |
| `vbat_mv` | `int` | Battery voltage. |
| `rssi` | `int` | The node's WiFi RSSI, for diagnosing coverage. |
| `fw` | `string` | Firmware version, so a bad build is identifiable in stored history. |

`plant_id` is compiled into the node.

QoS 0 means a lost sample is a gap rather than an incident, and retention means a restarting subscriber gets the last state of every plant immediately.

## Broker access

Each node needs a broker password and an entry in the [mqtt-broker](https://github.com/OscarCarPu/mqtt-broker) ACL granting write on `events/plants/<plant_id>/telemetry`. A user missing from the ACL connects fine and is then denied every topic, which looks exactly like a broken sensor.
