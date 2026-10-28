# plants-watcher

Battery soil-sensor nodes. Publish soil moisture, soil temp, air temp/RH to MQTT hourly.

- Producer only: ends at the broker. Everything downstream: [central-pipeline](https://github.com/OscarCarPu/central-pipeline).
- Nodes publish raw mV; no on-device decisions.

```
XIAO ESP32-C3 → WiFi → events/plants/<plant_id>/telemetry → central-pipeline
```

## Status
Planning, no code.

## Docs
- [hardware](docs/hardware.md)
- [topics_mqtt](docs/topics_mqtt.md)
- [roadmap](docs/roadmap.md)

## Layout (planned)
```
node/      Rust firmware
hardware/  KiCad
case/      OpenSCAD
```
