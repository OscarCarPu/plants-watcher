# plants-watcher

Battery-powered soil sensors that live in a plant pot and tell you when — and why — the plant needs water. Each node measures soil moisture, soil temperature and ambient temperature/humidity, and publishes them to MQTT once an hour.

This repo is a producer and nothing else: firmware, hardware and enclosure. It ends at the broker. [central-pipeline](https://github.com/OscarCarPu/central-pipeline) owns everything from MQTT onward — ingestion, storage, calibration, watering logic and alerts — so there is no resident process and no database here.

```
Node (XIAO ESP32-C3, battery)
  → WiFi → events/plants/<plant_id>/telemetry
  → central-pipeline
```

**Nodes do not decide.** They publish raw millivolts and nothing more. All calibration and watering logic lives downstream, so it can be changed and re-run against stored history without reflashing devices buried in pots.

## Status

Planning. No code yet. The first goal is one node on a breadboard with the hardware wired correctly — see the [Roadmap](docs/roadmap.md). The radio is decided later, from measured numbers.

## Docs

- [Hardware](docs/hardware.md) — parts, wake cycle, power, BOM
- [MQTT topics](docs/topics_mqtt.md) — the telemetry payload
- [Roadmap](docs/roadmap.md) — phases and exit criteria

## Planned layout

```
node/       Rust firmware — measure, publish, deep sleep
hardware/   KiCad
case/       OpenSCAD
```
