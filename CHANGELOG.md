# Changelog

## 1.1.0

**Breaking.** The reference is now always the bathroom's own 24 h median dew point.

- Removed the "house average" reference and the fixed-value option; removed the example house sensor. No sensors in other rooms are needed.
- `ref_dew` is replaced by the required `baseline_dew` input (a `statistics` helper with the 24 h median). `ref_manual` was removed.
- New default thresholds: ON 3.5 °C, OFF 2 °C above the baseline.
- New `cooldown` input (pause between cycles, default 15 min). After the pause, if the bathroom is still above the ON value, a new cycle starts (at most 3 in a row).
- Fix: the start trigger only fired when the condition turned true, so a shower was missed if the bathroom was already above the ON value (for example during quiet hours). The blueprint now also checks when quiet hours end and after every cycle.
- New examples: dew point from temperature and humidity, and the baseline helper. README rewritten.

## 1.0.0

First release. Reference dew point from a house-average sensor, the bathroom's own 24 h median, or a fixed value.
