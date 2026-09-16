# Weather Entity Input Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a Thermal Comfort device use a weather entity instead of two helper sensors, and attach forecast (especially `forecast_next`) to the same sensors.

**Architecture:** Same virtual device and sensor types. Weather mode reads current T/H from weather attributes and `weather.get_forecasts` for later periods. Index math is shared via `calculations.py`. No new entities, no forecast-type UI, no custom card.

**Tech Stack:** Home Assistant custom component (Python 3.11+, HA >= 2023.12.0), pytest-homeassistant-custom-component, voluptuous.

**Spec:** `docs/superpowers/specs/2026-09-16-weather-entity-design.md`

## Global Constraints

- Home Assistant >= 2023.12.0
- Python >= 3.11.0
- Do not change calculation formulas or existing sensor-mode unique IDs
- Weather entity **or** both T/H sensors, never both
- No forecast-type picker (hourly if supported, else daily, else twice-daily)
- English translations only
- Follow existing Home Assistant style in this repo

## File map

- Create: `custom_components/thermal_comfort/calculations.py`
- Create: `tests/test_calculations.py`, `tests/test_weather.py`
- Modify: `const.py`, `sensor.py`, `__init__.py`, `config_flow.py`, `translations/en.json`
- Modify: `tests/conftest.py`, `tests/const.py`, `tests/test_config_flow.py`
- Modify: `documentation/yaml.md`, `documentation/config_flow.md`

---

### Task 1: Weather entity as the current T/H source

This is the helper-sensor killer. Forecast comes in Task 2.

**Files:**
- Modify: `custom_components/thermal_comfort/const.py`
- Modify: `custom_components/thermal_comfort/sensor.py` (schema + `DeviceThermalComfort` weather listener)
- Modify: `custom_components/thermal_comfort/__init__.py`
- Modify: `custom_components/thermal_comfort/config_flow.py`
- Modify: `custom_components/thermal_comfort/translations/en.json`
- Modify: `tests/conftest.py`, `tests/const.py`, `tests/test_config_flow.py`
- Create: `tests/test_weather.py`

**Interfaces:**
- Consumes: existing `DeviceThermalComfort` current-value path
- Produces: `CONF_WEATHER_ENTITY`; weather XOR sensors validation; weather attribute reader; config form with optional weather entity

- [ ] **Step 1: Add constants**

In `custom_components/thermal_comfort/const.py`:

```python
CONF_WEATHER_ENTITY = "weather_entity"
ATTR_FORECAST = "forecast"
ATTR_FORECAST_NEXT = "forecast_next"
ATTR_FORECAST_NEXT_DATETIME = "forecast_next_datetime"
WEATHER_FORECAST_DAILY = 1
WEATHER_FORECAST_HOURLY = 2
WEATHER_FORECAST_TWICE_DAILY = 4
```

- [ ] **Step 2: Write failing tests**

In `tests/conftest.py`:

```python
WEATHER_ENTITY_ID = "weather.test"


def async_set_weather(
    hass,
    temperature: float = 25.0,
    humidity: float = 50.0,
    unit: str = UnitOfTemperature.CELSIUS,
    condition: str = "sunny",
    supported_features: int = 3,
) -> None:
    hass.states.async_set(
        WEATHER_ENTITY_ID,
        condition,
        {
            "temperature": temperature,
            "temperature_unit": unit,
            "humidity": humidity,
            "supported_features": supported_features,
        },
    )
```

Create `tests/test_weather.py` with YAML weather setup (same `name` / `unique_id` as `DEFAULT_TEST_SENSORS`, but `weather_entity: weather.test` instead of the two sensors). Fixture `start_ha_weather` is `start_ha` except it calls `async_set_weather` instead of `async_set_source_sensors`.

Tests:

- `test_weather_current_values`: absolute humidity state `"11.5128065738593"`, attributes temperature 25.0 and humidity 50.0
- `test_weather_fahrenheit_current_values`: set weather to 77 °F / 50 % RH, same absolute humidity, attribute temperature 77.0
- `test_weather_unavailable`: set weather state to `unavailable`, dew point becomes `STATE_UNAVAILABLE`
- `test_yaml_rejects_mixed_sources` / `test_yaml_requires_a_source`: `SENSOR_SCHEMA` raises `vol.Invalid`

Config flow tests (`tests/test_config_flow.py`):

- Existing `test_successful_config_flow` still works when submitting today's temperature+humidity fields (no weather key).
- New `test_weather_config_flow`: `async_set_weather`, submit `{name, weather_entity}` (plus advanced defaults if the form requires them). `CREATE_ENTRY`, `data` has `weather_entity`, does not have temperature/humidity sensors.
- New `test_mixed_sources_error`: submitting weather + both sensors returns form error `need_weather_or_sensors`.

- [ ] **Step 3: Run tests; they fail**

```bash
pytest tests/test_weather.py tests/test_config_flow.py::test_weather_config_flow -v
```

Expected: FAIL (`CONF_WEATHER_ENTITY` / schema still requires both sensors).

- [ ] **Step 4: YAML schema XOR**

In `sensor.py`, import `CONF_WEATHER_ENTITY` from const (define a local alias only if the file already aliases the other CONF keys that way). Replace `SENSOR_SCHEMA` so temperature/humidity/weather are all optional, then:

```python
def _validate_sensor_source(config: dict) -> dict:
    has_weather = CONF_WEATHER_ENTITY in config
    has_both_sensors = (
        CONF_TEMPERATURE_SENSOR in config and CONF_HUMIDITY_SENSOR in config
    )
    has_one_sensor = (
        CONF_TEMPERATURE_SENSOR in config
    ) ^ (
        CONF_HUMIDITY_SENSOR in config
    )
    if has_weather and (CONF_TEMPERATURE_SENSOR in config or CONF_HUMIDITY_SENSOR in config):
        raise vol.Invalid("Use weather_entity or temperature_sensor+humidity_sensor, not both")
    if has_weather or has_both_sensors:
        return config
    raise vol.Invalid("Provide weather_entity or both temperature_sensor and humidity_sensor")
```

`SENSOR_SCHEMA = vol.All(vol.Schema({...}).extend(SENSOR_OPTIONS_SCHEMA.schema), _validate_sensor_source)`

- [ ] **Step 5: DeviceThermalComfort weather listener**

Constructor gains `weather_entity: str | None = None`. If set, subscribe only to that entity; do not subscribe to T/H sensors.

```python
    async def _new_weather_state(self, state) -> None:
        if self._shutdown:
            return
        if state is None or state.state in (STATE_UNKNOWN, STATE_UNAVAILABLE):
            self._temperature = None
            self._humidity = None
            self.extra_state_attributes.pop(ATTR_TEMPERATURE, None)
            self.extra_state_attributes.pop(ATTR_HUMIDITY, None)
            await self.async_update_sensors(True)
            return

        unit = state.attributes.get(
            "temperature_unit", self.hass.config.units.temperature_unit
        )
        try:
            temp = util.convert(state.attributes.get("temperature"), float)
            temperature = TemperatureConverter.convert(
                temp, unit, UnitOfTemperature.CELSIUS
            )
        except (TypeError, ValueError):
            temperature = None
            temp = None
        try:
            humidity = float(state.attributes.get("humidity"))
        except (TypeError, ValueError):
            humidity = None

        temperature_ok = temperature is not None and -89.2 <= temperature <= 56.7
        humidity_ok = humidity is not None and 0 < humidity <= 100

        if temperature_ok:
            self._temperature = temperature
            self.extra_state_attributes[ATTR_TEMPERATURE] = temp
        else:
            self._temperature = None
            self.extra_state_attributes.pop(ATTR_TEMPERATURE, None)

        if humidity_ok:
            self._humidity = humidity
            self.extra_state_attributes[ATTR_HUMIDITY] = humidity
        else:
            self._humidity = None
            self.extra_state_attributes.pop(ATTR_HUMIDITY, None)

        if temperature_ok and humidity_ok:
            await self.async_update()
            return
        _LOGGER.info(
            "Weather entity %s is missing a valid temperature or humidity: %s",
            self._weather_entity,
            state,
        )
        await self.async_update_sensors(True)
```

Pass `weather_entity=data.get(CONF_WEATHER_ENTITY)` from `async_setup_entry` and `device_config.get(CONF_WEATHER_ENTITY)` from YAML setup. Store it in `hass.data` in `__init__.py` `async_setup_entry`.

- [ ] **Step 6: One-screen config flow**

Do **not** add an `input_source` step. Extend `build_schema`:

- Always include optional `CONF_WEATHER_ENTITY` selector `{entity: {domain: "weather"}}`.
- Temperature and humidity selectors become `vol.Optional` (keep defaults when lists are non-empty).
- If `config_entry` already has a weather entity, include it in the weather selector even if missing (same trick as T/H today).
- Abort only when there are **no** weather entities **and** no T/H sensors. If weather entities exist, show the form even with zero T/H sensors.

`check_input`:

```python
def check_input(hass: HomeAssistant, user_input: dict) -> dict:
    result = {}
    weather_id = user_input.get(CONF_WEATHER_ENTITY)
    temp_id = user_input.get(CONF_TEMPERATURE_SENSOR)
    hum_id = user_input.get(CONF_HUMIDITY_SENSOR)
    if weather_id and (temp_id or hum_id):
        result["base"] = "need_weather_or_sensors"
        return result
    if weather_id:
        if hass.states.get(weather_id) is None:
            result[CONF_WEATHER_ENTITY] = "weather_not_found"
        return result
    if not temp_id or not hum_id:
        result["base"] = "need_weather_or_sensors"
        return result
    if hass.states.get(temp_id) is None:
        result[CONF_TEMPERATURE_SENSOR] = "temperature_not_found"
    if hass.states.get(hum_id) is None:
        result[CONF_HUMIDITY_SENSOR] = "humidity_not_found"
    return result
```

On create, if weather: unique_id `weather-{registry unique_id or entity_id}`. If sensors: keep today's `{t}-{h}` unique_id. Strip empty weather/temp/humidity keys so they are not stored.

Options flow: if the entry has `weather_entity`, the form is name + weather + advanced (no T/H). Otherwise today's T/H form (no weather field). No mode switch.

English strings: `weather_entity`, `need_weather_or_sensors` ("Choose a weather entity or both a temperature and a humidity sensor"), `weather_not_found`. Do not add `input_source`.

- [ ] **Step 7: Run tests**

```bash
pytest tests/test_weather.py tests/test_config_flow.py tests/test_sensor.py tests/test_init.py -v
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add custom_components/thermal_comfort tests
git commit -m "$(cat <<'EOF'
feat: accept a weather entity as the current T/H source

Read temperature and humidity from weather attributes so outdoor
thermal comfort does not need helper sensors.
EOF
)"
```

---

### Task 2: Forecast on the same sensors

**Files:**
- Create: `custom_components/thermal_comfort/calculations.py`
- Create: `tests/test_calculations.py`
- Modify: `custom_components/thermal_comfort/sensor.py`
- Modify: `tests/test_weather.py`

**Interfaces:**
- Consumes: Task 1 weather listener; existing formulas in `DeviceThermalComfort`
- Produces: `calculate_sensor(sensor_type, temperature, humidity)`; `forecast` / `forecast_next` / `forecast_next_datetime` on weather-mode sensors

- [ ] **Step 1: Extract formulas**

Create `tests/test_calculations.py` asserting 25 °C / 50 % RH:

- `calculate_absolute_humidity` ≈ `11.5128065738593`
- `calculate_heat_index` ≈ `24.8611111111111`
- `calculate_dew_point_perception` returns `(DewPointPerception.COMFORTABLE, {dew_point: ...})`
- `calculate_sensor(SensorType.ABSOLUTE_HUMIDITY, 25, 50)` matches absolute humidity

Create `calculations.py`. Each current `DeviceThermalComfort` index method becomes `calculate_<name>(temperature, humidity)` with the **same body**, substituting `self._temperature` → `temperature`, `self._humidity` → `humidity`, and `await self.<other>()` → `calculate_<other>(...)`.

Avoid a circular import: `calculations.py` must not import `sensor.py` at module top. Import `SensorType`, perception enums, and `ATTR_*` **inside** `calculate_sensor` and inside any function that needs them.

```python
def calculate_sensor(sensor_type, temperature: float, humidity: float):
    from .sensor import SensorType

    calculators = {
        SensorType.ABSOLUTE_HUMIDITY: calculate_absolute_humidity,
        SensorType.DEW_POINT: calculate_dew_point,
        SensorType.DEW_POINT_PERCEPTION: calculate_dew_point_perception,
        SensorType.FROST_POINT: calculate_frost_point,
        SensorType.FROST_RISK: calculate_frost_risk,
        SensorType.HEAT_INDEX: calculate_heat_index,
        SensorType.HUMIDEX: calculate_humidex,
        SensorType.HUMIDEX_PERCEPTION: calculate_humidex_perception,
        SensorType.MOIST_AIR_ENTHALPY: calculate_moist_air_enthalpy,
        SensorType.RELATIVE_STRAIN_PERCEPTION: calculate_relative_strain_perception,
        SensorType.SUMMER_SCHARLAU_PERCEPTION: calculate_summer_scharlau_perception,
        SensorType.WINTER_SCHARLAU_PERCEPTION: calculate_winter_scharlau_perception,
        SensorType.SUMMER_SIMMER_INDEX: calculate_summer_simmer_index,
        SensorType.SUMMER_SIMMER_PERCEPTION: calculate_summer_simmer_perception,
        SensorType.THOMS_DISCOMFORT_PERCEPTION: calculate_thoms_discomfort_perception,
    }
    return calculators[sensor_type](temperature, humidity)
```

Replace each `DeviceThermalComfort` method body with:

```python
    @compute_once_lock(SensorType.DEW_POINT)
    async def dew_point(self) -> float:
        """Dew Point <http://wahiduddin.net/calc/density_algorithms.htm>."""
        from .calculations import calculate_dew_point

        return calculate_dew_point(self._temperature, self._humidity)
```

Do that for every index method. Run:

```bash
pytest tests/test_calculations.py tests/test_sensor.py -v
```

Expected: PASS with unchanged numeric strings in `test_sensor.py`.

- [ ] **Step 2: Write failing forecast tests**

In `start_ha_weather`, register `weather.get_forecasts` **before** setup, returning a list stored on `hass.data["test_weather_forecast_payload"]` (start empty). Tests that need data assign the list then `async_set_weather` to refresh.

Payload:

```python
FORECAST_HOURLY = [
    {"datetime": "2026-09-16T15:00:00+00:00", "temperature": 26.0, "humidity": 55},
    {"datetime": "2026-09-16T16:00:00+00:00", "temperature": 24.0, "humidity": 60},
    {"datetime": "2026-09-16T17:00:00+00:00", "temperature": 23.0},  # skip: no humidity
]
```

`test_weather_forecast_next`: after filling the payload and refreshing, heat index `forecast` has length 2, `forecast_next` ≈ `calculate_heat_index(26.0, 55.0)`, `forecast_next_datetime` is the first datetime.

`test_sensor_mode_has_no_forecast`: parametrize `DEFAULT_TEST_SENSORS`; `forecast` / `forecast_next` / `forecast_next_datetime` absent.

`test_forecast_is_unrecorded`: those three names are in `SensorThermalComfort._unrecorded_attributes`.

Run `pytest tests/test_weather.py::test_weather_forecast_next -v` — FAIL (attribute missing).

- [ ] **Step 3: Fetch and attach forecast**

On `DeviceThermalComfort`:

```python
    def _resolve_forecast_type(self, supported_features: int) -> str | None:
        if supported_features & WEATHER_FORECAST_HOURLY:
            return "hourly"
        if supported_features & WEATHER_FORECAST_DAILY:
            return "daily"
        if supported_features & WEATHER_FORECAST_TWICE_DAILY:
            return "twice_daily"
        return None

    async def async_refresh_forecasts(self) -> None:
        from .calculations import calculate_sensor

        if self._shutdown or not self._weather_entity:
            return
        state = self.hass.states.get(self._weather_entity)
        if state is None or state.state in (STATE_UNKNOWN, STATE_UNAVAILABLE):
            return
        forecast_type = self._resolve_forecast_type(
            int(state.attributes.get("supported_features", 0) or 0)
        )
        if forecast_type is None:
            self._forecasts = {sensor_type: [] for sensor_type in SENSOR_TYPES}
            return
        try:
            response = await self.hass.services.async_call(
                "weather",
                "get_forecasts",
                {"type": forecast_type},
                target={"entity_id": self._weather_entity},
                blocking=True,
                return_response=True,
            )
        except Exception:
            _LOGGER.warning(
                "Could not fetch forecasts from %s",
                self._weather_entity,
                exc_info=True,
            )
            return

        periods = list(
            (response or {}).get(self._weather_entity, {}).get("forecast") or []
        )
        unit = state.attributes.get(
            "temperature_unit", self.hass.config.units.temperature_unit
        )
        computed = {sensor_type: [] for sensor_type in SENSOR_TYPES}
        for period in periods:
            datetime_iso = period.get("datetime")
            try:
                temp = util.convert(period.get("temperature"), float)
                humidity = float(period.get("humidity"))
                temperature = TemperatureConverter.convert(
                    temp, unit, UnitOfTemperature.CELSIUS
                )
            except (TypeError, ValueError):
                continue
            if datetime_iso is None:
                continue
            if not -89.2 <= temperature <= 56.7 or not 0 < humidity <= 100:
                continue
            for sensor_type in SENSOR_TYPES:
                result = calculate_sensor(sensor_type, temperature, humidity)
                extras = {}
                value = result
                if isinstance(result, tuple) and len(result) == 2:
                    value, extras = result[0], dict(result[1])
                computed[sensor_type].append(
                    {
                        "datetime": datetime_iso,
                        "temperature": temp,
                        "humidity": humidity,
                        "value": value,
                        **extras,
                    }
                )
        self._forecasts = computed
        await self.async_update_sensors(True)
```

Call `async_refresh_forecasts` at the end of a successful `_new_weather_state` and from `_async_poll_update` when `self._weather_entity` is set. Do not clear forecasts when weather is briefly unavailable.

On `SensorThermalComfort`:

```python
    _unrecorded_attributes = frozenset(
        {ATTR_FORECAST, ATTR_FORECAST_NEXT, ATTR_FORECAST_NEXT_DATETIME}
    )
```

In `async_update`, after setting the current native value:

```python
        if self._device.weather_entity:
            forecast = self._device.forecasts.get(self._sensor_type, [])
            self._attr_extra_state_attributes[ATTR_FORECAST] = forecast
            if forecast:
                self._attr_extra_state_attributes[ATTR_FORECAST_NEXT] = forecast[0]["value"]
                self._attr_extra_state_attributes[ATTR_FORECAST_NEXT_DATETIME] = (
                    forecast[0]["datetime"]
                )
            else:
                self._attr_extra_state_attributes.pop(ATTR_FORECAST_NEXT, None)
                self._attr_extra_state_attributes.pop(ATTR_FORECAST_NEXT_DATETIME, None)
        else:
            self._attr_extra_state_attributes.pop(ATTR_FORECAST, None)
            self._attr_extra_state_attributes.pop(ATTR_FORECAST_NEXT, None)
            self._attr_extra_state_attributes.pop(ATTR_FORECAST_NEXT_DATETIME, None)
```

Add `weather_entity` and `forecasts` properties on the device. Init `self._forecasts = {}`.

- [ ] **Step 4: Run tests**

```bash
pytest tests/test_weather.py tests/test_sensor.py tests/test_config_flow.py tests/test_calculations.py tests/test_init.py -v
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add custom_components/thermal_comfort/calculations.py custom_components/thermal_comfort/sensor.py tests
git commit -m "$(cat <<'EOF'
feat: attach weather forecasts to thermal comfort sensors

Compute the same indices for forecast periods and expose the next
value plus the full list on the existing sensors.
EOF
)"
```

---

### Task 3: Documentation

**Files:**
- Modify: `documentation/yaml.md`, `documentation/config_flow.md`

- [ ] **Step 1: YAML docs**

Add the Outside `weather_entity` example from the spec. Document XOR. Document attributes: state is now; `forecast_next` is the next period (the useful one on perception sensors); `forecast` is the full list in More Info / automations.

Show one entities-card example (Now + attribute `forecast_next`). Do not add a Markdown table as a required dashboard. Mention the stock Weather forecast card cannot show these sensors.

- [ ] **Step 2: Config-flow docs**

Add: optional Weather entity field on the same form; pick that **or** the two sensors. Weather-only installs no longer need helper sensors.

- [ ] **Step 3: Commit**

```bash
git add documentation/yaml.md documentation/config_flow.md
git commit -m "$(cat <<'EOF'
docs: describe weather entity source and forecast_next

Document the XOR setup rule and that current sensors plus
forecast_next are the outdoor dashboard.
EOF
)"
```

---

## Plan self-review

1. **Spec coverage:** Weather as T/H source, XOR setup, auto forecast type, `forecast` + `forecast_next`, no wizard / no custom card / no expected-vs-actual.
2. **Placeholders:** Calculation bodies are a 1:1 move from `sensor.py` (same formulas). All names used later are defined in Task 1–2.
3. **Types:** `CONF_WEATHER_ENTITY`, `ATTR_FORECAST`, `ATTR_FORECAST_NEXT`, `ATTR_FORECAST_NEXT_DATETIME`, `calculate_sensor(...)` are consistent.
