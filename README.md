# plants-watcher

Battery-powered soil sensors that live in a plant pot and tell you when — and why — the plant needs water. Each node measures soil moisture, soil temperature and ambient temperature/humidity, and publishes them to MQTT once an hour.

This repo is a producer and nothing else: firmware, hardware and enclosure. It ends at the broker. [central-pipeline](https://github.com/OscarCarPu/central-pipeline) owns everything from MQTT onward — ingestion, storage, calibration, watering logic and alerts — so there is no resident process and no database here.

```
Node (XIAO ESP32-C3, battery)
  → WiFi → events/plants/<plant_id>/telemetry
  → central-pipeline: raw (bronze) → staging (silver) → marts (gold)
  → watering alerts, gv-api
```

## Status

Planning. No code yet — see the [Roadmap](docs/roadmap.md).

The radio is deliberately still open. Phases 0 and 1 run on a XIAO ESP32-C3 publishing MQTT directly over WiFi, because that gets real soil data into the pipeline in days rather than weeks: no PCB, no BLE bridge, no new toolchain. Phase 2 decides whether to stay there or move to nRF52840 + BLE, from measured numbers rather than estimates. The [MQTT contract](docs/topics_mqtt.md) is radio-agnostic by design, so nothing downstream depends on the answer.

**Nodes do not decide.** They publish raw millivolts and nothing more. All calibration and watering logic lives downstream, so it can be changed and re-run against stored history without reflashing devices buried in pots.

## Docs

- [Architecture](docs/architecture.md) — topology, node wake cycle, power budget
- [Hardware](docs/hardware.md) — sensing chain, the open radio decision, BOM and costs
- [MQTT topics](docs/topics_mqtt.md) — everything this repo publishes, and the contract downstream depends on
- [Watering logic](docs/watering_logic.md) — turning millivolts into "water this plant"
- [Roadmap](docs/roadmap.md) — phases, milestones and exit criteria

## Planned layout

```
node/       Rust firmware — measure, publish, deep sleep
hardware/   KiCad
case/       OpenSCAD
```

If phase 2 chooses BLE, the nodes stop speaking IP and a Go bridge is needed to translate. That bridge belongs in central-pipeline, not here — see [Bridging](docs/topics_mqtt.md#bridging-phase-2).
