# Hardware

Phases 0 and 1: one **XIAO ESP32-C3** on a breadboard, publishing MQTT directly over WiFi. No PCB.

It is chosen for speed, not as the final design: no PCB to design, onboard LiPo charging so a battery node needs no power path, and USB serial/JTAG built into the chip so there is no debug probe. A working node exists in days, and phase 0's real question — does the probe work in soil — gets answered before anything is committed.

## Parts

| Function | Part | Why |
|---|---|---|
| Soil moisture | Cheap capacitive probe, analogue output into the ADC | The riskiest part of the project. Validate it first. |
| Soil temperature | NTC 10k at the probe tip | Capacitive readings drift with temperature; this is what compensates them. No protocol, zero idle current. |
| Air temp + humidity | SHT40 (I2C) | +/-1.8 %RH, 0.08 uA idle, low long-term drift. |
| Battery | LiPo 1200 mAh on the XIAO's charger | Voltage is read through the ADC. |

The sensors sit on a switched rail so they draw nothing while the node sleeps.

## Wake cycle

1. Wake from deep sleep, every hour.
2. Enable the sensor rail, let it settle, read soil (averaged over N samples), NTC, SHT40 and battery, disable the rail.
3. Connect WiFi with a static IP and cached BSSID and channel; SNTP if the clock is unset.
4. Publish `events/plants/<id>/telemetry`, `seq += 1`.
5. Deep sleep.

Sensors are read before the radio comes up and the rail is off again before association starts, so the battery only ever supports the radio's peak draw.

Publishing is best-effort. A failed association or an unresponsive broker costs one sample and the node goes back to sleep — a retry loop turns a network problem into a flat battery. The gap shows up downstream as a missing `seq`.

The only state kept across cycles is the `seq` counter and the cached association parameters, both in RTC memory.

## Power

About 90 uAh per hourly cycle, split roughly half sleeping (~45 uA on the XIAO board) and half on WiFi (~80 mA for ~2 s). That is ~2.2 mAh/day, or about **18 months** on 1200 mAh.

Two levers, in order: association time (a static IP and cached BSSID is the difference between ~2 s and ~6 s awake) and sample interval (hourly is plenty; soil moisture moves over days). These are estimates until phase 1 measures them.

## BOM

Approximate, delivered to Spain, VAT excluded.

| Item | EUR |
|---|---|
| XIAO ESP32-C3 x2 (two probes side by side in phase 0) | 10 |
| Cheap capacitive probe x2 | 4 |
| SHT40 breakout x2 | 10 |
| NTC 10k, passives | 2 |
| LiPo 1200 mAh (phase 1) | 8 |
| **Total** | **34** |

Optionally add one commercial sensor (Tuya Zigbee or HHCC Flora, ~18 EUR) as a baseline to check readings against.

## Enclosure

- Vent the SHT40. A fully sealed case measures the humidity inside the case.
- Keep electronics above the soil. Avoid a cable-to-probe joint, since that is where water wicks in.
