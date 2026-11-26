# Bathroom extractor fan by dew point (Home Assistant blueprint)

Turns a bathroom extractor fan on when someone showers and off when the air is back to normal. It decides by comparing the **dew point of the bathroom** with a **reference dew point** that represents "what is normal here".

## Why dew point instead of relative humidity?

Relative humidity depends on temperature. During a shower the bathroom gets warmer *and* more humid, so RH ends up somewhere in the middle and then drops misleadingly fast once the room warms up, even though there is still plenty of steam. The dew point measures the actual amount of water vapour, so it is not affected by temperature and gives a much clearer signal (a shower typically raises it by 10 °C or more).

Comparing against a reference (instead of a fixed humidity threshold) also copes with houses that are humid in winter, where an absolute "start at 70 %" rule would run all the time.

## Features

- Starts when `bathroom dew point − reference` stays above a threshold for 1 minute. Optional absolute-humidity backup trigger.
- Stops when the difference stays below a lower threshold for N minutes, with a **minimum** and a hard **maximum** run time.
- **Only switches off cycles it started itself**: if you turn the fan on manually, it will not turn it off.
- **Quiet hours** (default 00:00–06:00): the fan will not start. A cycle already running is not interrupted.
- **Optional window sensor**: while the window is open the fan is not started.
- **Fail-safe**: a lost sensor ends the cycle and switches the fan off; a reference sensor that is `unavailable` prevents starting; after a Home Assistant restart the fan is switched off.
- No notifications. Works with any `switch` (smart relay) and any bathroom sensor that reports dew point and humidity.

## Requirements

1. A switch entity controlling the fan.
2. A bathroom sensor with **dew point** and **humidity** entities. If your sensor does not report dew point, create a template sensor with the same Magnus formula used in [`examples/house_reference_dew_point.yaml`](examples/house_reference_dew_point.yaml).
3. A **reference dew point**, in one of three ways (see below).

## The reference dew point: three options

The blueprint compares the bathroom with a reference. Pick **one** of these, from best to worst. If you select a sensor it is always used; the fixed value is only used when the sensor field is left empty.

### Option 1 (best): average dew point of the rest of the house

Needs temperature **and** humidity sensors in a few other rooms (not bathrooms). It adapts to the weather and the season, and several bathrooms can share one sensor. See [`examples/house_reference_dew_point.yaml`](examples/house_reference_dew_point.yaml): edit the list of rooms and add it to your configuration, then select it as the reference sensor.

### Option 2: the bathroom's own 24 h median (no other sensors needed)

If you have no sensors elsewhere, use the bathroom itself as its own baseline: the **median** of its dew point over the last 24 hours. A shower lasts minutes, so the median barely moves and represents the bathroom's normal state. It adapts to the season by itself and also absorbs the fact that a bathroom usually sits a bit warmer and wetter than the rest of the house.

**The blueprint cannot compute this itself**: a blueprint has no history, so you need a separate `statistics` sensor. Create it once per bathroom:

- *UI:* Settings → Devices & services → Helpers → Create helper → **Statistics**. Entity: the bathroom dew point sensor. Statistic: **Arithmetic median** (`median`). Maximum age: **24 hours**. Sampling size: **1000**.
- *YAML:* see [`examples/bathroom_dew_point_baseline.yaml`](examples/bathroom_dew_point_baseline.yaml).

Then select that new sensor as the **reference sensor** in the blueprint.

Things to know:

- The statistics sensor reads Home Assistant's recorder history on start-up, so it is usually available immediately. While it has no value, the fan will **not** start (fail-safe).
- If the bathroom is humid for more than about half of the day (fan broken, very heavy use), the median creeps up and detection gets less sensitive. The maximum run time still protects you.
- Because the resting offset is about zero with this option, you can usually use **smaller thresholds** than the defaults, which were chosen for option 1. Tune from real showers.

### Option 3 (last resort): a fixed value in °C

Leave the reference sensor empty and type a dew point in the **Fixed reference dew point** field. It works, but read the problems below first.

**Problems of a fixed value**

- **It drifts with the seasons.** The normal dew point of a home can be around 6–10 °C in winter and 15–20 °C in summer. A value that is right today will be wrong in a few months.
- **Too low a value → the fan runs without showers.** The bathroom already looks "above normal" on any humid summer day, so it starts (and keeps running until the maximum time).
- **Too high a value → showers are missed.** In winter the bathroom never reaches the start threshold, so the fan never starts.
- **Weather is not the only variable.** Cooking, drying laundry, an open door or a different number of people at home all move the real baseline, and a fixed number cannot follow them.
- **It hides the resting offset.** The bathroom normally sits above the rest of the house; with a fixed value you have to account for this yourself or the "off" threshold may never be reached, so the fan only stops at the maximum time.
- **Nothing warns you.** If the number becomes wrong, the symptoms (fan always on, fan never on) are easy to blame on something else.

If you must use it: measure the bathroom's dew point when nobody has used it for hours, put that value in, keep generous thresholds, **revisit it every season**, and keep the humidity backup trigger enabled. Better: use option 2, which costs one helper and no extra sensors.

## Installation

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FDidacChaves%2Fha-bathroom-fan-blueprint%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fbathroom_fan_dew_point.yaml)

Or copy `blueprints/automation/bathroom_fan_dew_point.yaml` to `config/blueprints/automation/<any-folder>/` and reload.

Then create one automation per bathroom from the blueprint.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| Extractor fan | – | `switch` that controls the fan |
| Bathroom dew point / humidity | – | Bathroom sensor entities |
| Reference dew point sensor | empty | Optional. Option 1 or 2 above. If empty, the fixed value is used |
| Fixed reference dew point | 10 °C | Option 3. Only used when no reference sensor is selected |
| Window sensor | none | Optional; `on` = open, fan will not start |
| Difference to turn ON | 8 °C | Bathroom minus reference |
| Difference to turn OFF | 5 °C | Must hold for the "off" delay |
| Emergency absolute humidity | 80 % | Starts the fan regardless of the reference |
| Minutes below threshold before off | 3 | |
| Minimum / maximum run time | 10 / 60 min | Maximum is a safety limit |
| Quiet hours | 00:00–06:00 | No automatic start in this window |

## Tuning

Defaults are estimates. With option 1 a bathroom often sits a bit warmer and wetter than the house even at rest (e.g. +1.5 °C), so the *off* threshold must be above that resting offset or the fan will only stop at the maximum time. Watch a few real showers (graph the bathroom dew point, the reference and their difference) and adjust per bathroom. If the fan starts and stops repeatedly, make the *off* threshold a bit lower than the *on* threshold.

## Safety notes

The blueprint switches a mains-powered device through your own relay; make sure that installation is safe and rated for the fan. The maximum run time and restart behaviour are there to avoid a fan left on indefinitely, but this is provided as is, without warranty.
