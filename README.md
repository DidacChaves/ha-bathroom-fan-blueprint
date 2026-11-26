# Bathroom extractor fan by dew point (Home Assistant blueprint)

Turns a bathroom extractor fan on when someone showers and off when the air is back to normal. It decides by comparing the **dew point of the bathroom** with the **average dew point of the rest of the house**.

## Why dew point instead of relative humidity?

Relative humidity depends on temperature. During a shower the bathroom gets warmer *and* more humid, so RH ends up somewhere in the middle and then drops misleadingly fast once the room warms up, even though there is still plenty of steam. The dew point measures the actual amount of water vapour, so it is not affected by temperature and gives a much clearer signal (a shower typically raises it by 10 °C or more).

Comparing against the rest of the house (instead of using a fixed threshold) also copes with houses that are humid in winter, where an absolute "start at 70 %" rule would run all the time.

## Features

- Starts when `bathroom dew point − house dew point` stays above a threshold for 1 minute. Optional absolute-humidity backup trigger.
- Stops when the difference stays below a lower threshold for N minutes, with a **minimum** and a hard **maximum** run time.
- **Only switches off cycles it started itself**: if you turn the fan on manually, it will not turn it off.
- **Quiet hours** (default 00:00–06:00): the fan will not start. A cycle already running is not interrupted.
- **Optional window sensor**: while the window is open the fan is not started.
- **Fail-safe**: a lost sensor ends the cycle and switches the fan off; after a Home Assistant restart the fan is switched off.
- No notifications and no helpers needed. Works with any `switch` (smart relay) and any bathroom sensor that reports dew point and humidity.

## Requirements

1. A switch entity controlling the fan.
2. A bathroom sensor with **dew point** and **humidity** entities. If your sensor does not report dew point, create a template sensor with the same Magnus formula used in the example.
3. A **house reference dew point** sensor. See [`examples/house_reference_dew_point.yaml`](examples/house_reference_dew_point.yaml): edit the list of rooms and add it to your configuration. Several bathrooms can share the same one.

## Installation

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FDidacChaves%2Fha-bathroom-fan-blueprint%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fbathroom_fan_dew_point.yaml)

Or copy `blueprints/automation/bathroom_fan_dew_point.yaml` to `config/blueprints/automation/<any-folder>/` and reload.

Then create one automation per bathroom from the blueprint.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| Extractor fan | – | `switch` that controls the fan |
| Bathroom dew point / humidity | – | Bathroom sensor entities |
| House reference dew point | – | Average dew point of the rest of the house |
| Window sensor | none | Optional; `on` = open, fan will not start |
| Difference to turn ON | 8 °C | Bathroom minus house |
| Difference to turn OFF | 5 °C | Must hold for the "off" delay |
| Emergency absolute humidity | 80 % | Starts the fan regardless of the reference |
| Minutes below threshold before off | 3 | |
| Minimum / maximum run time | 10 / 60 min | Maximum is a safety limit |
| Quiet hours | 00:00–06:00 | No automatic start in this window |

## Tuning

Defaults are estimates. A bathroom often sits a bit warmer and wetter than the house even at rest (e.g. +1.5 °C), so the *off* threshold must be above that resting offset or the fan will only stop at the maximum time. Watch a few real showers (graph the bathroom dew point, the house reference and their difference) and adjust per bathroom.

## Safety notes

The blueprint switches a mains-powered device through your own relay; make sure that installation is safe and rated for the fan. The maximum run time and restart behaviour are there to avoid a fan left on indefinitely, but this is provided as is, without warranty.
