# Data models

Records are what the cabinet sends and what the server stores. Setpoints are local to the cabinet. They are not required on the server for the loop to run.

Times are ISO-8601 in UTC. The cabinet clock is the source of `recorded_at`. The server adds `received_at`. Ids are strings chosen by the cabinet so a retry does not create a second row.

## Cabinet

| Field | Type | Meaning |
| --- | --- | --- |
| `cabinet_id` | string | Stable id of this unit |
| `name` | string | Human label |
| `fw` | string | Software version on the Pi |

One cabinet, one pot. No pot id until there is a second pot.

## Reading

A reading is one observation. The cabinet may send several in one POST.

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | Unique per attempt. Reused on retry |
| `cabinet_id` | string | |
| `recorded_at` | timestamp | When the Pi took the sample |
| `kind` | enum | `moisture`, `temperature`, `npk`, `light`, `float`, `vision` |
| `value` | object | Shape depends on `kind` |

`value` by kind:

| Kind | Fields |
| --- | --- |
| `moisture` | `raw` (ADC counts), `volts` |
| `temperature` | `celsius` |
| `npk` | `n`, `p`, `k` in the probe's own units. No conversion is invented here |
| `light` | `on` (bool), `duty` (0–1, PCA9685 channel 0) |
| `float` | `water_present` (bool). `true` means the tank is not empty |
| `vision` | Later. `label`, `confidence`, `frame_at`. Labels: `leaf_green`, `leaf_yellow`, `leaf_brown`, `spots`, `bloom`, `bud` |

Absent kinds are omitted. NPK is not read on every cycle, and vision does not exist until the camera is fitted.

## Dose

A dose is a pump pulse or a light change the loop decided to make.

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | Unique per attempt. Reused on retry |
| `cabinet_id` | string | |
| `recorded_at` | timestamp | When the pulse started |
| `channel` | enum | `water`, `a`, `b`, `light` |
| `seconds` | number | How long the channel was driven. For light, the new duty is in `duty` and `seconds` may be omitted |
| `duty` | number | Light only. 0–1 |
| `reason` | enum | Why |

Reasons:

| Reason | Use |
| --- | --- |
| `moisture_low` | Water pulse |
| `n_short` | Pump A |
| `p_short`, `k_short` | Pump B |
| `in_range` | Not sent. No dose is recorded for a skipped pulse |
| `schedule` | Light on or a planned duty change |
| `too_warm` | Light backed off |
| `float_empty` | Water, A, and B held |
| `daily_cap` | Water held |
| `sensor_fault` | Related channel held |

## Fault

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | string | |
| `cabinet_id` | string | |
| `recorded_at` | timestamp | |
| `source` | enum | `ads1115`, `npk`, `pca9685`, `float`, `temperature`, `camera` |
| `detail` | string | Short, not a stack trace |

## Aggregate

Computed on the server. Not sent by the cabinet.

| Field | Type | Meaning |
| --- | --- | --- |
| `cabinet_id` | string | |
| `bucket` | timestamp | Start of the hour or the day |
| `grain` | enum | `hour`, `day` |
| `moisture_avg` | number | Mean of `raw`, if any samples |
| `temperature_min` | number | |
| `temperature_max` | number | |
| `light_on_seconds` | number | From light doses and duty |
| `water_seconds` | number | Sum of water pulses |
| `a_seconds` | number | |
| `b_seconds` | number | |
| `npk_last` | object | Last `n`, `p`, `k` in the bucket |
| `vision_counts` | object | Later. Count per label |

A bucket with no samples is not written.

## Local setpoint

Kept on the Pi. Listed so the loop and the models use the same names.

| Field | Meaning |
| --- | --- |
| `light_on` | Local time the bar turns on |
| `light_off` | Local time the bar turns off |
| `light_duty` | Duty while on, before thermal backoff |
| `temp_limit_c` | Back off the light above this |
| `moisture_low` | ADC raw below this starts a drink |
| `moisture_high` | ADC raw above this stops a drink |
| `water_pulse_s` | Length of one water pulse |
| `water_wait_s` | Pause before the next moisture read |
| `water_cap_s` | Max water-on time per local day |
| `n_low`, `p_low`, `k_low` | Dose below these probe units |
| `a_pulse_s`, `b_pulse_s` | Length of one concentrate pulse |
| `npk_period_h` | Hours between NPK reads |
| `nutrient_cooldown_h` | Minimum hours between concentrate pulses |
