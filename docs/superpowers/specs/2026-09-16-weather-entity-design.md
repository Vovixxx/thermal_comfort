# Weather Entity Input — Design Spec

**Date:** 2026-09-16
**Status:** Draft for implementation
**Related:** [dolezsa/thermal_comfort#231](https://github.com/dolezsa/thermal_comfort/issues/231), [dolezsa/thermal_comfort#412](https://github.com/dolezsa/thermal_comfort/issues/412)

## What this integration already does

Thermal Comfort takes **temperature + humidity** and expands them into extra sensors: dew point, frost risk, heat index, and the text perceptions people actually glance at ("comfortable", "a bit dry for some", "probable frost").

That is useful indoors. Outdoors, the same T+H already live on a `weather.*` entity (Met.no, Open-Meteo, and so on). Today you have to copy those two attributes into helper sensors, then feed the helpers into this integration. The helpers are busywork. And even after that, you only get **now** — not "what will it feel like later." Many weather providers never ship these comfort texts at all. That is the gap this feature fills.

## Simple product

Same virtual device. Same sensors. Two changes:

1. **You can point it at a weather entity** instead of two helpers. Current T+H come from the weather attributes. The sensors you already know keep working.
2. **Those same sensors also know the next stretch of weather.** Each one keeps a forecast list, and a single `forecast_next` value so "what to expect" is one attribute, especially on the text sensors.

No new sensor types. No extra `*_forecast` entities. No custom card. No fake weather entity. Indoor devices stay exactly as they are.

```text
weather.forecast_home          indoor T + RH sensors
        │                              │
        ▼                              ▼
   Thermal Comfort                Thermal Comfort
   virtual device                 virtual device
        │                              │
        ▼                              ▼
   dew point, frost risk,         same sensors,
   "comfortable", …               current only
   (now + forecast_next)
```

## What we are not doing

- Helper sensors (the point is to delete them)
- Mixing weather + T/H sensors on one device
- Switching an existing device from sensors to weather in options (create a new device)
- A forecast-type picker — pick hourly if the weather entity has it, otherwise daily
- Custom Lovelace cards, a `weather` platform, or expected-vs-actual tracking (interesting later, not this change)
- Changing any calculation formula

## Setup (one form, same as today)

Keep the current config screen. Add one optional field:

- **Weather entity** (optional)
- Temperature sensor and humidity sensor stay on the form

Rules:

- Weather entity **or** both sensors, not both, not neither.
- Unique ID for weather devices: `weather-{weather unique_id}` (fallback `weather-{entity_id}`).
- Sensor devices keep `{temp unique_id}-{humidity unique_id}`.
- Config entry version stays 2. Missing `weather_entity` means today's sensor mode.
- Options flow shows the same source the device was created with.

YAML:

```yaml
thermal_comfort:
  - sensor:
    - name: Outside
      weather_entity: weather.forecast_home
      unique_id: 7c2e0b5a-4d11-4f0c-9c3e-weather-outside
    - name: Living Room
      temperature_sensor: sensor.temperature_livingroom
      humidity_sensor: sensor.humidity_livingroom
      unique_id: 2f842c63-051a-4c49-9da2-4f04ee677514
```

If there are no T/H sensors but there is a weather entity, the form still opens (today it aborts). If there is neither, abort as today.

## Current values from weather

A weather entity's **state** is `sunny` / `clear-night`. Temperature and humidity are **attributes**. That is why pointing the existing temperature picker at `weather.home` fails.

Read:

- `temperature` + `temperature_unit` (convert to °C, same range check as now)
- `humidity` (same 0–100 check as now)

Unavailable weather → sensors unavailable, same as a dead indoor sensor.

## Forecast

On weather-entity updates (and on poll if polling is on), call `weather.get_forecasts`. Use hourly if `supported_features` includes it, else daily, else twice-daily. No user setting.

For each period that has both temperature and humidity, run the same calculations we already run for "now." Skip incomplete periods.

Each weather-mode sensor then has:

| Piece | What it is |
| --- | --- |
| **state** | Current value (number or perception text). This is what you already put on a dashboard. |
| **`forecast_next`** | The next period's value. The convenient "what to expect." |
| **`forecast_next_datetime`** | When that next value is for. |
| **`forecast`** | Full list: `{datetime, temperature, humidity, value, …}` for automations and More Info. |

If there is no usable forecast, omit `forecast_next` / `forecast_next_datetime` and set `forecast` to `[]`.

Sensor-mode devices do not get these attributes.

Mark `forecast`, `forecast_next`, and `forecast_next_datetime` unrecorded so they do not fill the database.

A forecast fetch failure must not blank the current sensors. Keep the last good forecast and log a warning.

Do not invent expected-vs-actual history in this change. Current sensor history already records what actually happened; forecast attributes are the prediction.

## How you actually use it

You already know the indoor pattern: put `dew_point_perception` and `frost_risk` on a dashboard. Outdoor weather mode is the same cards, fed by Met.no / Open-Meteo instead of helpers.

**Now** is the sensor state. **Next** is `forecast_next` — one extra row on an entities card, or one attribute in an automation (`if frost_risk forecast_next is high`). The text sensors are the ones that matter most here.

The full `forecast` list is there when you open More Info or when a template/automation wants the whole horizon. That is enough. We are not building a forecast strip.

## Calculations

Share one implementation between current and forecast. Move the existing formulas into `calculations.py` as pure functions `(temperature_c, humidity) -> value`. Wrappers on `DeviceThermalComfort` stay so current behavior does not change. Formulas must not change; existing numeric tests are the contract.

## Errors

| Situation | Behavior |
| --- | --- |
| Weather entity missing at config time | Field error `weather_not_found` |
| Form submitted with both weather and sensors, or neither | Form error `need_weather_or_sensors` |
| Weather missing at runtime | Sensors unavailable |
| Period missing humidity or temperature | Skip that period |
| `get_forecasts` fails | Warning; keep last forecast; current values stay |
| Duplicate weather unique ID | Abort `already_configured` |

## Tests (minimum)

- Weather current values match today's 25 °C / 50 % RH numbers (including a °F weather entity).
- Unavailable weather → sensors unavailable.
- YAML and config flow reject mixed/missing sources; weather-only setup works; existing sensor flow still works.
- Forecast list skips incomplete periods; `forecast_next` equals `forecast[0].value`.
- Sensor-mode sensors have no forecast attributes.
- Existing `tests/test_sensor.py` numbers do not change.

## Docs and translations

- `documentation/yaml.md` and `documentation/config_flow.md`: weather field, XOR rule, `forecast` / `forecast_next`.
- English strings only in `en.json`.

## Constraints

- Home Assistant >= 2023.12.0 (`weather.get_forecasts` already exists at this floor)
- Python >= 3.11.0
- Do not change calculation formulas or existing sensor-mode unique IDs
