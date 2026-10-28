# Phase 0 — Probe validation

## Setup
- [ ] Hardware list for v0.0.1 ([hardware](hardware.md))
- [ ] Breadboard: XIAO + probe + NTC + SHT40

## Firmware (throwaway)
- [ ] Read probe, NTC, SHT40, vbat
- [ ] Publish `events/plants/<id>/telemetry` every 5 min ([topics](topics_mqtt.md))

## Pipeline
- [ ] central-pipeline ingests into `raw.mqtt_events`

## Checklist 
- [ ] Calibration and checks for the sensors
- [ ] 24 hours of static soil for noise 
- [ ] 1 week of watering and leting dry, no plant
- [ ] 2 weeks on a plant with normal watering patters
- [ ] comparison and modeling of the two probes and the static soil for noise

# Hardware List 
- ESP32c3 (for now, revisit on battery management)
- NTC 10K B3435 (check which ones theres at home first)
- SHT40 Adafruit sensirion STEMMA QT (check which one its at home first)
- SEN0193 at start (if corrosion hits a lot coat in epoxy or use SEN0308)
