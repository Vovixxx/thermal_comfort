# Weather Entity Input — Design Spec

**Date:** 2026-09-16
**Status:** Draft for implementation
**Related:** [dolezsa/thermal_comfort#231](https://github.com/dolezsa/thermal_comfort/issues/231) (weather entity as input), [dolezsa/thermal_comfort#412](https://github.com/dolezsa/thermal_comfort/issues/412) (`weather.home` cannot be used as a temperature/humidity sensor)

## Problem

Thermal Comfort currently requires two `sensor` entities: a temperature sensor and a humidity sensor. Home Assistant weather entities already expose both values as **state attributes** (`temperature`, `humidity`) and can also provide **forecasts** via `weather.get_forecasts`.

Two gaps follow from that:

1. **Current values cannot be read from a weather entity.** The calculator treats the source entity *state* as a float. A weather entity's state is a condition string (`sunny`, `clear-night`), so setup with `weather.home` logs "invalid value" and never produces sensors ([#412](https://github.com/dolezsa/thermal_comfort/issues/412)).
2. **Projected (forecast) thermal comfort is unavailable.** Users who want tomorrow's heat index, frost risk, or dew-point perception must build template sensors from forecast data themselves ([#231](https://github.com/dolezsa/thermal_comfort/issues/231)).

This feature adds a first-class weather-entity source that produces the same thermal-comfort sensors as today, plus projected values from the weather forecast.

## Goals

- A user can create a Thermal Comfort device from a single `weather.*` entity.
- That device produces the **current** thermal-comfort sensors using the weather entity's current `temperature` and `humidity` attributes.
- Each of those sensors also exposes **projected** values for every forecast period that includes both temperature and humidity.
- Existing temperature-plus-humidity config entries, YAML, unique IDs, and calculation results stay unchanged.
- Sensor-based and weather-based sources are mutually exclusive on one device.

## Non-goals

- Combining a weather entity with extra temperature/humidity sensors on the same device.
- Switching an existing sensor-based config entry to weather (or the reverse) in the options flow.
- New sensor types, new thermal indices, or wind-chill / wet-bulb formulas.
- Using weather `dew_point` or `apparent_temperature` instead of calculating our own.
- Recording forecast attributes in the recorder (forecasts are future-valued and high-cardinality).
- Fetching forecasts more often than the weather entity updates (or the existing poll interval).

## Approaches considered

### A. Weather source + forecast attributes on existing sensors (recommended)

Add an optional weather-entity source. Current sensors stay the same entities (`dew_point`, `heat_index`, …). Each sensor gains a `forecast` extra state attribute: a list of `{datetime, temperature, humidity, value, …}` dicts.

- **Pros:** No entity explosion (still one virtual device and the same 16 sensor types). Automations and custom cards can read `state_attr('sensor.outside_heat_index', 'forecast')`. Matches how weather entities themselves used to expose forecasts.
- **Cons:** Native HA history graphs cannot plot the future; users who want a chart use a custom card or a template.

### B. Weather source for current values only

Read `temperature` / `humidity` attributes from a weather entity and stop there.

- **Pros:** Smallest change; fixes #412.
- **Cons:** Does not deliver projected values, which is half of this feature and the main reason #231 asked for weather support.

### C. Extra forecast entities per index

Create additional sensors such as `heat_index_forecast` whose state is the next period and whose attributes are the rest of the horizon.

- **Pros:** Slightly easier to put "next hour's heat index" on a dashboard as a state.
- **Cons:** Doubles entity count; unique-id and enablement UI get more complex; still needs an attribute list for the full horizon. YAGNI relative to A.

**Decision:** Approach A.

## Architecture

```text
                    ┌─────────────────────────┐
                    │  weather.forecast_home  │
                    │  state: sunny           │
                    │  attrs: temperature,    │
                    │         humidity,       │
                    │         temperature_unit│
                    └───────────┬─────────────┘
                                │
          state_changed         │  weather.get_forecasts
                                │
                    ┌───────────▼─────────────┐
                    │  DeviceThermalComfort   │
                    │  current: T, RH         │
                    │  forecasts: list[T, RH] │
                    └───────────┬─────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
        dew_point          heat_index      frost_risk  …
        state: current     state: current  state: current
        forecast: [...]    forecast: [...] forecast: [...]
```

Two input modes, one compute device:

| Mode | Current T / RH | Projected T / RH |
| --- | --- | --- |
| Sensors (existing) | Source sensor states | None (`forecast` omitted or `[]`) |
| Weather (new) | Weather entity attributes | `weather.get_forecasts` periods that include both temperature and humidity |

Index math is extracted into pure functions in `calculations.py` so current and forecast paths share one implementation. `DeviceThermalComfort` keeps the Home Assistant lifecycle (listeners, polling, shutdown, availability).

## Configuration

### Config flow

Two-step UI. Step one is always available, even when no temperature/humidity sensors exist (so weather-only installs can set up the integration).

1. **`user`:** `name`, `input_source` (`sensors` or `weather`). `input_source` is flow-only and is **not** stored on the config entry.
2. **`sensors`:** existing temperature + humidity selectors, plus advanced options. Unique ID remains `{temperature_unique_id}-{humidity_unique_id}`.
3. **`weather`:** weather-entity selector (`domain: weather`) and `forecast_type` (`auto`, `hourly`, `daily`, `twice_daily`). Unique ID is `weather-{weather_unique_id}` (fallback: `weather-{entity_id}`). Abort if that unique ID is already configured.

Advanced options (`poll`, `scan_interval`, `custom_icons`, `enabled_sensors`) appear on the second step, same as today.

**Options flow:** follow the mode already stored on the entry. Sensor entries keep today's schema. Weather entries show weather entity + forecast type + advanced options. No mode switch.

**Validation:** the weather entity must exist as a state. Missing current `temperature` / `humidity` does not block setup; sensors stay unavailable until both attributes are numeric and in range (same policy as missing source sensors).

### YAML

A device is valid if it has **either** `weather_entity` **or** both `temperature_sensor` and `humidity_sensor`. Mixing them on one device is a config error.

```yaml
thermal_comfort:
  - sensor:
    - name: Outside
      weather_entity: weather.forecast_home
      forecast_type: auto   # optional, default auto
      unique_id: 7c2e0b5a-4d11-4f0c-9c3e-weather-outside
    - name: Living Room
      temperature_sensor: sensor.temperature_livingroom
      humidity_sensor: sensor.humidity_livingroom
      unique_id: 2f842c63-051a-4c49-9da2-4f04ee677514
```

`forecast_type` is ignored for sensor-mode devices.

### Stored config keys

| Key | Required | Default | Notes |
| --- | --- | --- | --- |
| `name` | yes | | Existing |
| `temperature_sensor` | sensor mode | | Existing |
| `humidity_sensor` | sensor mode | | Existing |
| `weather_entity` | weather mode | | New |
| `forecast_type` | weather mode | `auto` | New: `auto` \| `hourly` \| `daily` \| `twice_daily` |
| `poll`, `scan_interval`, `custom_icons`, `enabled_sensors` | no | existing defaults | Unchanged |

Config entry `VERSION` stays **2**. Weather keys are additive; missing `weather_entity` means sensor mode.

## Current values from weather

Weather entities store measurements as attributes, not as state.

- Temperature: `state.attributes["temperature"]`, unit from `state.attributes["temperature_unit"]` (fallback: `hass.config.units.temperature_unit`). Convert to Celsius with `TemperatureConverter`, then apply the existing range check `-89.2 … 56.7`.
- Humidity: `state.attributes["humidity"]`, range `0 < humidity <= 100` (existing).
- Invalid / `unknown` / `unavailable` weather state: `_temperature` and `_humidity` become `None`, sensors become `available=False` (existing lifecycle).
- `extra_state_attributes["temperature"]` stores the value in the weather entity's presentation unit, matching how sensor mode stores the source sensor's native value.
- Subscribe with `async_track_state_change_event` on the weather entity; unsubscribe in `async_shutdown` (already used by the runtime device).

## Projected values from forecasts

### Fetching

Call Home Assistant's `weather.get_forecasts` service (`blocking=True`, `return_response=True`) with `type` set to the resolved forecast type. This API exists in Home Assistant >= 2023.12.0, which is already this integration's minimum.

Refresh forecasts:

- when the weather entity state changes, and
- on the poll interval when `poll: true`.

If the service fails or returns no list, log a warning and keep the last successful forecast. Never make current sensors unavailable because a forecast fetch failed.

### Resolving `forecast_type`

Read `supported_features` from the weather entity state (bit flags: daily=1, hourly=2, twice_daily=4).

- `auto`: hourly if supported, else twice-daily, else daily, else no forecasts (`[]`).
- Explicit type: use it when the matching flag is set; otherwise `[]` and a warning. Current sensors still work.

### Computing a period

For each forecast dict:

1. Skip if `temperature` or `humidity` is missing or not numeric.
2. Convert temperature to Celsius using the weather entity's `temperature_unit`.
3. Apply the same range checks as current values; skip out-of-range periods.
4. For daily / twice-daily periods, use `temperature` (the period high). Do not average with `templow`.
5. Run every enabled sensor type's pure calculation on that (T, RH) pair.

### Attribute shape

Each thermal-comfort sensor adds:

```yaml
forecast:
  - datetime: "2026-09-16T15:00:00+00:00"
    temperature: 26.0          # weather presentation unit
    humidity: 55
    value: 24.12               # this sensor's native value (or perception string)
    # plus the same extra keys the current sensor already exposes, e.g. dew_point
```

`datetime` is copied from the weather forecast (`datetime` key). Periods without `datetime` are skipped.

Device-level extra attribute `forecast_type` records the resolved type (`hourly` / `daily` / `twice_daily`) or is omitted when no forecast is available.

Sensor-mode devices do not set `forecast` (no attribute, not an empty list).

### Recorder

Mark `forecast` as unrecorded on `SensorThermalComfort` via `_unrecorded_attributes = frozenset({"forecast"})` so hourly lists do not inflate the database.

## Calculation extraction

Move the bodies of `DeviceThermalComfort` index methods into `custom_components/thermal_comfort/calculations.py` as synchronous functions:

```python
def calculate_dew_point(temperature: float, humidity: float) -> float: ...
def calculate_dew_point_perception(temperature: float, humidity: float) -> tuple[DewPointPerception, dict]: ...
```

A dispatcher `calculate_sensor(sensor_type, temperature, humidity)` is used by the forecast loop.

Existing async methods become one-line wrappers behind `compute_once_lock`. Formulas must not change; existing numeric tests remain the contract.

## Error handling

| Situation | Behavior |
| --- | --- |
| Weather entity missing at config time | Form error `weather_not_found` |
| Weather entity missing at runtime | Sensors unavailable |
| Weather has no current humidity/temperature | Sensors unavailable; log info (same tone as invalid source sensors) |
| Forecast period missing humidity | Skip that period |
| `get_forecasts` raises / empty | Warning; keep last forecast; current values unchanged |
| Requested forecast type unsupported | Warning; `forecast` empty; current values unchanged |
| YAML mixes weather and sensors | Voluptuous invalid config |
| Duplicate weather unique ID | Abort `already_configured` |

## Testing

- Unit tests for `calculations.py` against known values already asserted in `tests/test_sensor.py` (e.g. 25 °C / 50 % RH → absolute humidity `11.5128065738593`).
- Existing YAML/config-entry sensor tests continue to pass with no formula drift.
- New tests: weather current values (including °F conversion), unavailable weather, YAML exclusive source, config flow `user` → `weather`, forecast attribute contents, skipped periods, `auto` preferring hourly, unrecorded attribute, sensor-mode sensors have no `forecast` key.
- Mock `weather.get_forecasts` by registering a test service; do not require a real weather platform.

## Documentation and translations

- Update `documentation/yaml.md` and `documentation/config_flow.md`.
- Add English strings in `custom_components/thermal_comfort/translations/en.json`. Other locales are filled later by Fink / inlang; Home Assistant falls back to English.

## Compatibility constraints

- Home Assistant >= 2023.12.0
- Python >= 3.11.0
- Do not change calculation formulas
- Do not change unique IDs of existing sensor-mode entities
- Follow Home Assistant development guidelines already used by this repo
