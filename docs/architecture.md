# Architecture

## Topology

```mermaid
graph LR
    subgraph pots["Pots (battery, 2xAAA)"]
        N1["Node 1"]
        N2["Node 2"]
        NN["Node N"]
    end

    subgraph homelab["Home Lab"]
        GW["BLE Gateway\n(Go, passive scan)"]
        MQTT["MQTT Broker"]
        Consumer["Consumer\n(Go)"]
        DB[("SQLite")]
    end

    Bridge["ESP32 BLE bridge\n(optional, mains powered)"]
    Telegram["Telegram Bot"]

    N1 -- "BLE adv" --> GW
    N2 -- "BLE adv" --> GW
    NN -- "BLE adv" --> Bridge
    Bridge -- "MQTT" --> MQTT
    GW -- "plants/+/telemetry" --> MQTT
    MQTT --> Consumer
    Consumer --> DB
    Consumer -- "events/plants/+/*" --> MQTT
    Consumer -. "alerts" .-> Telegram
```

The optional ESP32 bridge exists only for coverage. Indoor BLE range through walls is roughly 10-15 m, so a single gateway may not hear every pot. A mains-powered ESP32 placed in a far room decodes the same advertisements and publishes to the same topics — the consumer cannot tell the difference.

## Node — per wake cycle

The node runs the same cycle forever. Nothing persists across cycles except a sequence counter and the sensor warm-up state, both held in retained RAM.

```mermaid
flowchart TD
    Wake([RTC wakeup, every SAMPLE_INTERVAL]) --> PwrOn[Enable sensor rail]
    PwrOn --> Settle[Wait for probe oscillator to settle]
    Settle --> Soil["Drive probe with PWM\nRectify, read SAADC\nAverage N samples"]
    Soil --> NTC[Read soil NTC via SAADC]
    NTC --> SHT[Read SHT40 over I2C]
    SHT --> Vdd["Read VDD via SAADC\n(internal, no divider)"]
    Vdd --> PwrOff[Disable sensor rail]
    PwrOff --> Pack["Pack BTHome v2 payload\nseq += 1"]
    Pack --> Adv["Broadcast 3 advertisements\n~15 ms total"]
    Adv --> Sleep[System ON sleep with RTC]
    Sleep --> Wake
```

### Why System ON sleep, not System OFF

System OFF on the nRF52840 is ~0.4 uA versus ~1.5-3 uA for System ON with the RTC running. Over five years the difference is under 100 mAh out of a 1000 mAh AAA pair — real, but not decisive.

System ON keeps RAM and the RTC alive, which means the sequence counter and any calibration state survive without touching retention registers, and the Embassy timer just works. The simpler firmware is worth the microamp.

## Gateway

Passive BLE scanning on Linux requires kernel >= 5.10 and BlueZ >= 5.56. Passive means the gateway never sends a scan request, so nodes are never woken or asked to respond — they stay pure broadcasters.

The gateway holds the only piece of per-plant configuration it needs: a mapping from BLE MAC address to `plant_id`. Everything else about a plant lives in the consumer.

Its job is deliberately thin: decode BTHome, stamp `ts` on reception, look up the `plant_id`, publish. No logic, no storage. If it dies, you lose samples but nothing is corrupted.

## Consumer

```mermaid
flowchart TD
    Boot([Boot]) --> Load[Load plants.yaml]
    Load --> Connect["Connect MQTT\n(auto-reconnect)"]
    Connect --> Sub[Subscribe plants/+/telemetry]

    Sub --> Rx([Telemetry received])
    Rx --> Dedup{"seen (plant_id, seq)\nrecently?"}
    Dedup -->|Yes| Drop[Drop duplicate]
    Dedup -->|No| Store[Insert into SQLite]
    Store --> Calc["Convert raw_mv to moisture %\nusing per-plant calibration"]
    Calc --> Decide{Below watering threshold?}
    Decide -->|No| Idle[idle]
    Decide -->|Yes| Alerted{Already alerted?}
    Alerted -->|Yes| Idle
    Alerted -->|No| Notify["Telegram: 'Plant X needs water'\nPublish events/plants/X/alert"]
```

Deduplication by `(plant_id, seq)` is what makes multiple gateways safe. The same physical advertisement heard by two receivers produces two MQTT messages with the same sequence number; the consumer keeps the first.

The alert latch clears when moisture rises back above the threshold — same pattern as `ALERT_STATE` in mutual-watchdog, just held in the consumer instead of RTC memory.

## Power budget

Per node, at a 10 minute sample interval.

| Item | Current | Duration | Per cycle |
|---|---|---|---|
| Sleep (System ON + RTC) | ~2 uA | 600 s | 0.33 uAh |
| Sensor rail + measurement | ~1 mA | ~50 ms | 0.014 uAh |
| BLE advertising (3x) | ~5 mA | ~15 ms | 0.021 uAh |
| **Total** | | | **~0.37 uAh** |

That is ~0.9 uAh/hour, ~21 uAh/day, dominated almost entirely by sleep current. A 2xAAA alkaline pair (~1000 mAh) gives a theoretical life far beyond the shelf life of the batteries themselves; assume you are limited by self-discharge, not by the node.

The practical consequence: **do not optimise the radio, optimise the sleep current.** Anything left powered — a pull-up on an unused line, the probe oscillator, a leaky FET — costs more than every transmission combined.

For comparison, the same duty cycle over WiFi + MQTT costs roughly 150 uAh per cycle, four orders of magnitude more, which is why the radio choice is what it is. See [hardware](hardware.md#why-not-esp32--wifi).
