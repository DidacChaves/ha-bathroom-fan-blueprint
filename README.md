# Bathroom extractor fan by dew point (Home Assistant blueprint)

Turns a bathroom extractor fan on when someone showers and off when the air is back to normal. It decides by comparing the **dew point of the bathroom** with **that bathroom's own normal level** (its 24 h median). One bathroom sensor is all you need: no sensors in other rooms.

## Why dew point instead of relative humidity?

Relative humidity depends on temperature. During a shower the bathroom gets warmer *and* more humid, so RH ends up somewhere in the middle and then drops misleadingly fast once the room warms up, even though there is still plenty of steam. The dew point measures the actual amount of water vapour, so it is not affected by temperature and gives a much clearer signal.

## Why compare the bathroom with itself?

Every bathroom is its own microclimate: warmer, differently ventilated, door open or closed. Comparing it with other rooms mixes in differences that have nothing to do with showering and gives a small margin between "someone showered" and "normal drift". The bathroom's **own 24 h median** adapts to the bathroom, the weather and the season by itself, so the same thresholds mean the same thing in every bathroom.

The median is the value the bathroom is below half of the time and above the other half. A shower keeps the dew point high for minutes (up to about an hour without ventilation), so even 2–3 showers a day leave it far below 50 % of the day and the median barely moves. It would only be fooled by a bathroom that stays humid for more than about 12 hours a day.

## How it works

1. You create a **`statistics` helper** (24 h median of the bathroom dew point). A blueprint has no history, so this has to be a separate sensor, one per bathroom. See [Setup](#setup).
2. The fan starts when `bathroom dew point − baseline` reaches the **ON** value for 1 minute (or when humidity reaches an emergency level).
3. It stops when the rise stays below the **OFF** value for a few minutes, after a minimum run time and never later than the maximum run time.
4. It then stays off for a pause. If the bathroom is still above the ON value afterwards, another cycle starts (at most 3 in a row).

## Features

- **Only switches off cycles it started itself**: if you turn the fan on manually, it will not turn it off.
- **Quiet hours** (default 00:00–06:00): the fan will not start. A cycle already running is not interrupted. When quiet hours end the blueprint checks again, so a bathroom that is still humid gets its cycle.
- **Optional window sensor**: while the window is open the fan is not started.
- **Fail-safe**: a lost sensor ends the cycle and switches the fan off; an `unavailable` baseline prevents starting; after a Home Assistant restart the fan is switched off.
- No notifications and no polling: it reacts to sensor changes plus a re-check when quiet hours end and after each cycle.

## Requirements

- A **switch** entity controlling the fan (for example a smart relay).
- A bathroom sensor with **dew point** and **humidity** entities. If yours reports only temperature and humidity, use [`examples/bathroom_dew_point_from_temp_humidity.yaml`](examples/bathroom_dew_point_from_temp_humidity.yaml) to create the dew point.
- The **baseline helper** described below.

## Setup

### 1. Create the baseline helper (once per bathroom)

Settings → Devices & services → Helpers → Create helper → **Statistics**:

- Entity: the bathroom dew point sensor
- Statistic: **Arithmetic median** (`median`)
- Maximum age: **24 hours**
- Sampling size: **1000**

Home Assistant reads the recorder history on start-up, so the helper is usually useful immediately. While it has no value the fan will not start (fail-safe). A YAML equivalent is in [`examples/bathroom_dew_point_baseline.yaml`](examples/bathroom_dew_point_baseline.yaml).

### 2. Import the blueprint

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FDidacChaves%2Fha-bathroom-fan-blueprint%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fbathroom_fan_dew_point.yaml)

Or copy `blueprints/automation/bathroom_fan_dew_point.yaml` to `config/blueprints/automation/<any-folder>/` and reload.

### 3. Create one automation per bathroom

Select the fan switch, the bathroom dew point and humidity sensors, and the baseline helper from step 1.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| Extractor fan | – | `switch` that controls the fan |
| Bathroom dew point / humidity | – | Bathroom sensor entities |
| Baseline (24 h median) | – | The `statistics` helper from step 1 |
| Window sensor | none | Optional; `on` = open, fan will not start |
| Rise to turn ON | 3.5 °C | Bathroom dew point above the baseline |
| Rise to turn OFF | 2 °C | Must hold for the off delay; keep it below the ON value |
| Emergency absolute humidity | 80 % | Starts the fan regardless of the baseline |
| Minutes below OFF before off | 3 | |
| Minimum / maximum run time | 10 / 60 min | Maximum is a safety limit |
| Pause between cycles | 15 min | Then a new cycle starts only if still above ON |
| Quiet hours | 00:00–06:00 | No automatic start in this window |

## Tuning

The defaults come from one bathroom, so treat them as a starting point. In that bathroom the dew point stayed within roughly −1.5 to +1.1 °C of its 24 h median at rest, and showers rose about 4 to 5 °C above it. With ON = 3.5 °C and OFF = 2 °C that gave one cycle per shower and no false starts.

To tune a bathroom, graph its dew point together with the baseline helper for a few days:

- **Starts without a shower** → raise ON.
- **Misses short showers** → lower ON (it must stay above the resting noise of your bathroom).
- **Runs until the maximum time** → raise OFF (it must be above the level the dew point settles at after a shower).
- **Starts and stops repeatedly** → keep OFF clearly below ON, or increase the pause.

If the fan is not connected yet, the dew point falls on its own, more slowly than it will with ventilation. A threshold that works without the fan will work at least as well with it.

## Upgrading from 1.0

Version 1.1 is a **breaking change**:

- The "reference dew point sensor" (`ref_dew`) became the required `baseline_dew` input. Create the median helper and select it.
- The "house average" option and the fixed-value option (`ref_manual`) were removed. The example house sensor is gone. Dew point relative to other rooms gave a small margin, and a fixed value drifts with the seasons.
- New `pause between cycles` input, and the start now also re-checks when quiet hours end and after each cycle.
- Review your ON/OFF values: they were relative to the house, now they are relative to the bathroom's own median (defaults 3.5 / 2).

Version 1.0 remains available under [Releases](../../releases/tag/v1.0.0).

## Safety notes

The blueprint switches a mains-powered device through your own relay; make sure that installation is safe and rated for the fan. The maximum run time and restart behaviour are there to avoid a fan left on indefinitely, but this is provided as is, without warranty.
