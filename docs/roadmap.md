# Roadmap

Start a phase only after the previous exit criteria are met.

## Phase 0 — Validate probe

See [probe](probe.md).

## Phase 1 — One node end to end

- [ ] Rust firmware: wake → measure → WiFi → publish → sleep
- [ ] 30-min cycles
- [ ] Run on LiPo
- [ ] LiPo warning battery, battery calculations, power readings
- [ ] Small decision model for watering manually

Exit:
- 7 days of hourly readings on LiPo

## Later
- Radio: WiFi vs BLE
- PCB + enclosure
- Agronomy → central-pipeline
- Auto watering
