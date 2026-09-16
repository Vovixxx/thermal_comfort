# Stage 1 — Weather entity as temperature/humidity source

**Date:** 2026-09-16  
**Status:** Ready to implement  
**Parent:** `docs/superpowers/specs/2026-09-16-thermal-comfort-roadmap.md`  
**Related:** [dolezsa/thermal_comfort#231](https://github.com/dolezsa/thermal_comfort/issues/231), [dolezsa/thermal_comfort#412](https://github.com/dolezsa/thermal_comfort/issues/412)

## Problem

Thermal Comfort needs a temperature entity and a humidity entity. A `weather.*` entity already has both, as **attributes** (`temperature`, `humidity`). The integration currently treats source **state** as a float, so `weather.home` (state = `sunny`) never calculates.

The workaround is two helper sensors that copy those attributes. That is extra entities for no extra meaning.

## Stage 1 product

Add an optional **weather entity** field. If set, current T and H come from that entity's attributes. **Separate temperature and humidity sensors stay fully supported** — weather is another source, not a replacement for the sensor path. The integration still creates the same virtual device and the same comfort sensors as today.

Typical use later: one **global** outdoor device (weather *or* outdoor T+H sensors) plus room devices on indoor sensors. Stage 1 does not wire rooms to that global; it only makes weather a valid source so you can stop using helper sensors.

This stage does **not** attach sensors to the weather device, does **not** rename from the weather device, does **not** mark a device as global, and does **not** compute forecasts.

```text
weather.forecast_home  ──►  Thermal Comfort virtual device  ──►  dew point, frost risk, "comfortable", …
        (attributes T, H)           (unchanged model)
```

Indoor T+H devices keep working exactly as they do now.

## Setup

Same config form as today, plus optional **Weather entity**.

- Provide a weather entity **or** both sensors, not both, not neither.
- Name still defaults to `Thermal Comfort`, still editable (device-name inheritance is Stage 2).
- Unique ID for weather devices: `weather-{weather unique_id}` (fallback `weather-{entity_id}`).
- Sensor devices keep `{temp unique_id}-{humidity unique_id}`.
- Config entry version stays 2. Missing `weather_entity` = today's mode.
- Options: edit the same source type the entry was created with. No mode switch (create a new entry).
- If the install has a weather entity but no T/H sensors, the form still opens. If it has neither, abort as today.

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

## Runtime

Read weather **attributes**, not state:

- `temperature` + `temperature_unit` → convert to °C, existing range `-89.2 … 56.7`
- `humidity` → existing `0 < h <= 100`
- Store `extra_state_attributes["temperature"]` in the weather entity's presentation unit (same idea as sensor mode)

Unavailable / missing T or H → comfort sensors unavailable, same as a dead indoor sensor.

Subscribe with `async_track_state_change_event` on the weather entity; unsubscribe in existing `async_shutdown`.

Do not call `weather.get_forecasts` in this stage.

## Errors

| Situation | Behavior |
| --- | --- |
| Weather entity missing at config | Field error `weather_not_found` |
| Both weather and sensors, or neither | Form error `need_weather_or_sensors` |
| Weather missing at runtime | Sensors unavailable |
| Duplicate weather unique ID | Abort `already_configured` |

## Tests

- Weather at 25 °C / 50 % RH matches existing absolute humidity `"11.5128065738593"`
- Weather in °F (77 °F / 50 %) matches the same calculated Celsius results; attribute temperature stays 77
- Unavailable weather → comfort sensors `unavailable`
- YAML and config flow reject mixed/missing sources; weather-only create works; existing sensor flow still works
- `tests/test_sensor.py` numbers unchanged

## Docs and translations

- `documentation/yaml.md`, `documentation/config_flow.md`
- English only: `weather_entity`, `need_weather_or_sensors`, `weather_not_found`

## Constraints

- Home Assistant >= 2023.12.0, Python >= 3.11.0
- Do not change formulas or existing sensor-mode unique IDs
- Do not extract `calculations.py` until Stage 3 needs it
- Do not add forecast attributes, extra entities, or device-registry linking
- Do not add window/clothing advice sensors
