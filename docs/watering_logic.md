# Watering logic

The node emits millivolts. Millivolts mean nothing on their own: the same soil at the same water content reads differently depending on substrate, probe insertion depth, temperature and the individual board. This document is about closing that gap, and it is where the actual value of the project sits.

None of it is implemented in this repo. It lives in [central-pipeline](https://github.com/OscarCarPu/central-pipeline), downstream of the broker, so it can be changed and re-run against stored history without reflashing anything buried in a pot. This document is the intent; the models that implement it belong in central-pipeline's `docs/sources/plants.md`.

The levels below map onto the medallion layers: calibration is a silver-layer concern, everything derived from history is gold.

## Levels

### Level 0 — fixed threshold

"Alert if `raw_mv` below X." Works on the bench, fails across pots. Not used.

### Level 1 — two-point calibration (v1, silver)

At install time, record two references per node:

- `dry_mv` — probe in air
- `wet_mv` — probe in a glass of water

Then `moisture_pct = (dry_mv - raw_mv) / (dry_mv - wet_mv) * 100`, clamped.

This is not volumetric water content and does not pretend to be. It is a stable, comparable number per node, which is enough to alert on.

Temperature compensation applies here too: the capacitive reading drifts with soil temperature, so `raw_mv` is corrected against `soil_temp_c` using a coefficient measured in phase 0.

Both belong in the staging model, alongside the dedup on (`plant_id`, `seq`) — bronze keeps the millivolts exactly as they arrived, silver is where they become a percentage.

### Level 2 — MAD from observed history (phase 3, gold)

The pot itself tells you its own endpoints, so stop guessing them.

- A watering event — a sharp rise within one sample interval — settles a few hours later at that pot's **field capacity**.
- The lowest point reached before each watering is that pot's practical **dry point**.

Between the two you have the available water for *this* pot with *this* substrate and *this* plant, and you alert on **MAD** (Management Allowed Depletion): water when some fraction of it, typically 30-50 %, has been used.

This is what irrigation practice actually does, it needs no knowledge of the substrate, and it self-corrects when a plant is repotted.

### Level 3 — drying rate and prediction (phase 3, gold)

Fit the slope of the last N hours to get a depletion rate in percent per day, and report days remaining rather than a binary alert. The rate itself is a function of VPD and light, which is what makes the ambient sensors worth having: it explains *why* a plant is drying faster this week.

VPD comes from `air_temp_c` and `air_rh`. DLI comes from integrating `lux` over the day, once the VEML7700 is populated.

An hourly sample interval is more than enough for all three levels. Watering shows up as a step change between consecutive samples either way, and a drying rate fitted over days does not improve with denser points — which is the argument that keeps the node's radio budget affordable.

## Derived signals

Cheap to compute once history exists, and more useful than the raw alert:

| Signal | Detection | Meaning |
|---|---|---|
| **Watering detected** | Rise beyond a delta within one interval | Enables "you last watered this 12 days ago" |
| **Poor absorption** | Post-watering level returns to baseline within hours | Compacted or hydrophobic substrate — repot |
| **Never dries** | Sustained high moisture over N days | Root rot risk, or a pot with no drainage |
| **Probe drift** | Post-watering peak trending down month over month | The probe is absorbing water and needs replacing |
| **Sensor stale** | No telemetry for N intervals | Dead battery, or out of radio range |

**Probe drift** is the one worth building early. It is the known failure mode of every cheap capacitive sensor, and detecting it turns a silent wrong answer into an alert. The instinct is the one behind every watchdog in the home lab: a thing that monitors should also notice when it has stopped being trustworthy.

## Calibration workflow

1. Before installing, log `dry_mv` in air and `wet_mv` in water.
2. Insert the probe to root depth, away from the pot wall — readings near the wall are dominated by the air gap.
3. Water normally and let the system observe a full cycle.
4. From phase 3, the observed field capacity and dry point supersede the manual pair, which becomes only a sanity bound.

## Per-plant configuration

Calibration belongs with the logic, not with the node, so it lives in central-pipeline as reference data the staging models join against — a dbt seed, versioned in git, which makes a recalibration a commit and a rerun rather than a database edit nobody can reconstruct later.

| Column | Type | Description |
| ------ | ---- | ----------- |
| `plant_id` | `TEXT` | Matches the `plant_id` in the topic. **PK** |
| `species` | `TEXT` | Free text, for the alert message. |
| `pot_litres` | `NUMERIC` | Context for interpreting drying rate. |
| `substrate` | `TEXT` | Free text, e.g. `peat + perlite`. |
| `dry_mv` | `INT` | Probe in air, at install. |
| `wet_mv` | `INT` | Probe in water, at install. |
| `threshold_pct` | `NUMERIC` | Alert below this. Superseded by MAD in phase 3. |

The node holds none of this. Its entire configuration is its own `plant_id`, its WiFi credentials and its sample interval.
