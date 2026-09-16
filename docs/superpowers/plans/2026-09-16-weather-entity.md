# Stage 1 Implementation Plan — Weather entity as T/H source

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a Thermal Comfort device use a weather entity for **current** temperature and humidity instead of two helper sensors.

**Architecture:** Same virtual device, same sensors, same formulas. If `weather_entity` is set, read `temperature` / `humidity` attributes from that entity. No forecasts, no device-registry linking, no `calculations.py`.

**Tech Stack:** Home Assistant custom component (Python 3.11+, HA >= 2023.12.0), pytest-homeassistant-custom-component, voluptuous.

**Spec:** `docs/superpowers/specs/2026-09-16-weather-entity-design.md`  
**Roadmap:** `docs/superpowers/specs/2026-09-16-thermal-comfort-roadmap.md`

## Global Constraints

- Home Assistant >= 2023.12.0
- Python >= 3.11.0
- Do not change calculation formulas or existing sensor-mode unique IDs
- Weather entity **or** both T/H sensors, never both
- English translations only
- Follow existing Home Assistant style in this repo
- Do not implement later stages (device linking, forecasts, global/zones, windows/doors, clothing advice)

## File map

- Modify: `custom_components/thermal_comfort/const.py`
- Modify: `custom_components/thermal_comfort/sensor.py`
- Modify: `custom_components/thermal_comfort/__init__.py`
- Modify: `custom_components/thermal_comfort/config_flow.py`
- Modify: `custom_components/thermal_comfort/translations/en.json`
- Modify: `tests/conftest.py`, `tests/const.py`, `tests/test_config_flow.py`
- Create: `tests/test_weather.py`
- Modify: `documentation/yaml.md`, `documentation/config_flow.md`

---

### Task 1: Constants, helpers, failing tests

**Files:**
- Modify: `custom_components/thermal_comfort/const.py`
- Modify: `tests/conftest.py`
- Create: `tests/test_weather.py`

**Interfaces:**
- Consumes: existing `start_ha` / `async_set_source_sensors` pattern
- Produces: `CONF_WEATHER_ENTITY`; `WEATHER_ENTITY_ID`; `async_set_weather()`

- [ ] **Step 1: Add the config key**

In `custom_components/thermal_comfort/const.py` append:

```python
CONF_WEATHER_ENTITY = "weather_entity"
```

- [ ] **Step 2: Weather test helper**

In `tests/conftest.py` (next to `async_set_source_sensors`):

```python
WEATHER_ENTITY_ID = "weather.test"


def async_set_weather(
    hass,
    temperature: float = 25.0,
    humidity: float = 50.0,
    unit: str = UnitOfTemperature.CELSIUS,
    condition: str = "sunny",
) -> None:
    """Create a weather entity with current temperature and humidity attributes."""
    hass.states.async_set(
        WEATHER_ENTITY_ID,
        condition,
        {
            "temperature": temperature,
            "temperature_unit": unit,
            "humidity": humidity,
        },
    )
```

- [ ] **Step 3: Write failing weather tests**

Create `tests/test_weather.py`:

```python
"""Tests for weather-entity input (current values only)."""
import pytest
from voluptuous.error import Invalid

from custom_components.thermal_comfort.const import CONF_WEATHER_ENTITY, DOMAIN
from custom_components.thermal_comfort.sensor import SENSOR_SCHEMA, SensorType
from homeassistant.components.sensor import DOMAIN as PLATFORM_DOMAIN
from homeassistant.const import ATTR_TEMPERATURE, STATE_UNAVAILABLE, UnitOfTemperature
from homeassistant.core import HomeAssistant
from homeassistant.setup import async_setup_component
from pytest_homeassistant_custom_component.common import assert_setup_component

from .conftest import WEATHER_ENTITY_ID, async_set_weather
from .test_sensor import get_sensor

WEATHER_YAML = [
    "domains, config",
    [
        (
            [(DOMAIN, 1)],
            {
                DOMAIN: {
                    PLATFORM_DOMAIN: {
                        "name": "test_thermal_comfort",
                        CONF_WEATHER_ENTITY: WEATHER_ENTITY_ID,
                        "unique_id": "unique_thermal_comfort_id",
                    },
                },
            },
        ),
    ],
]


@pytest.fixture
async def start_ha_weather(hass, domains, config):
    """Set up thermal_comfort from a weather entity."""
    async_set_weather(hass)
    await hass.async_block_till_done()
    for domain, count in domains:
        with assert_setup_component(count, domain):
            assert await async_setup_component(hass, domain, config)
        await hass.async_block_till_done()
    await hass.async_start()
    await hass.async_block_till_done()


@pytest.mark.parametrize(*WEATHER_YAML)
async def test_weather_current_values(hass: HomeAssistant, start_ha_weather) -> None:
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).state == "11.5128065738593"
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).attributes[ATTR_TEMPERATURE] == 25.0
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).attributes["humidity"] == 50.0


@pytest.mark.parametrize(*WEATHER_YAML)
async def test_weather_fahrenheit_current_values(
    hass: HomeAssistant, start_ha_weather
) -> None:
    async_set_weather(
        hass, temperature=77.0, humidity=50.0, unit=UnitOfTemperature.FAHRENHEIT
    )
    await hass.async_block_till_done()
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).state == "11.5128065738593"
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).attributes[ATTR_TEMPERATURE] == 77.0


@pytest.mark.parametrize(*WEATHER_YAML)
async def test_weather_unavailable_makes_sensors_unavailable(
    hass: HomeAssistant, start_ha_weather
) -> None:
    hass.states.async_set(WEATHER_ENTITY_ID, "unavailable")
    await hass.async_block_till_done()
    assert get_sensor(hass, SensorType.DEW_POINT).state == STATE_UNAVAILABLE


def test_yaml_rejects_mixed_sources() -> None:
    with pytest.raises(Invalid):
        SENSOR_SCHEMA(
            {
                "name": "Mixed",
                CONF_WEATHER_ENTITY: WEATHER_ENTITY_ID,
                "temperature_sensor": "sensor.temp",
                "humidity_sensor": "sensor.hum",
                "unique_id": "mixed",
            }
        )


def test_yaml_requires_a_source() -> None:
    with pytest.raises(Invalid):
        SENSOR_SCHEMA({"name": "None", "unique_id": "none"})
```

- [ ] **Step 4: Run tests; they fail**

```bash
pytest tests/test_weather.py -v
```

Expected: FAIL (`CONF_WEATHER_ENTITY` not accepted by `SENSOR_SCHEMA` / setup).

- [ ] **Step 5: Commit**

```bash
git add custom_components/thermal_comfort/const.py tests/conftest.py tests/test_weather.py
git commit -m "test: add failing coverage for weather entity as T/H source"
```

---

### Task 2: YAML schema and weather attribute reader

**Files:**
- Modify: `custom_components/thermal_comfort/sensor.py`
- Modify: `custom_components/thermal_comfort/__init__.py`

**Interfaces:**
- Consumes: `CONF_WEATHER_ENTITY`
- Produces: XOR source validation; `DeviceThermalComfort(weather_entity=...)`; `_new_weather_state`

- [ ] **Step 1: Relax YAML schema**

Import `CONF_WEATHER_ENTITY` from const (or alias it in `sensor.py` the same way `CONF_TEMPERATURE_SENSOR` is aliased today). Replace `SENSOR_SCHEMA` so T, H, and weather are optional, then validate:

```python
def _validate_sensor_source(config: dict) -> dict:
    """Require weather_entity XOR both temperature and humidity sensors."""
    has_weather = CONF_WEATHER_ENTITY in config
    has_temperature = CONF_TEMPERATURE_SENSOR in config
    has_humidity = CONF_HUMIDITY_SENSOR in config
    if has_weather and (has_temperature or has_humidity):
        raise vol.Invalid(
            "weather_entity cannot be combined with temperature_sensor or humidity_sensor"
        )
    if has_weather or (has_temperature and has_humidity):
        return config
    raise vol.Invalid(
        "Provide weather_entity or both temperature_sensor and humidity_sensor"
    )


SENSOR_SCHEMA = vol.All(
    vol.Schema(
        {
            vol.Optional(CONF_NAME): cv.string,
            vol.Optional(CONF_TEMPERATURE_SENSOR): cv.entity_id,
            vol.Optional(CONF_HUMIDITY_SENSOR): cv.entity_id,
            vol.Optional(CONF_WEATHER_ENTITY): cv.entity_id,
            vol.Optional(CONF_ICON_TEMPLATE): cv.template,
            vol.Optional(CONF_ENTITY_PICTURE_TEMPLATE): cv.template,
            vol.Required(CONF_UNIQUE_ID): cv.string,
        }
    ).extend(SENSOR_OPTIONS_SCHEMA.schema),
    _validate_sensor_source,
)
```

- [ ] **Step 2: Weather listener on DeviceThermalComfort**

Constructor adds `weather_entity: str | None = None`. Store `self._weather_entity`. If it is set, subscribe only to that entity (reuse `_unsub_callbacks`). If not, keep today's T/H subscriptions.

```python
    async def weather_state_listener(self, event) -> None:
        if self._shutdown:
            return
        await self._new_weather_state(event.data.get("new_state"))

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
            temp = None
            temperature = None
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

Pass `weather_entity=` from `async_setup_platform` (`device_config.get(CONF_WEATHER_ENTITY)`) and `async_setup_entry` (`data.get(CONF_WEATHER_ENTITY)`).

In `__init__.py` `async_setup_entry`, store `CONF_WEATHER_ENTITY: get_value(entry, CONF_WEATHER_ENTITY)` next to the existing keys.

- [ ] **Step 3: Run weather + existing sensor tests**

```bash
pytest tests/test_weather.py tests/test_sensor.py tests/test_init.py -v
```

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add custom_components/thermal_comfort/sensor.py custom_components/thermal_comfort/__init__.py
git commit -m "$(cat <<'EOF'
feat: read current T/H from a weather entity

Allow YAML devices to use weather_entity instead of helper sensors.
EOF
)"
```

---

### Task 3: Config flow and docs

**Files:**
- Modify: `custom_components/thermal_comfort/config_flow.py`
- Modify: `custom_components/thermal_comfort/translations/en.json`
- Modify: `tests/test_config_flow.py`
- Modify: `documentation/yaml.md`, `documentation/config_flow.md`

**Interfaces:**
- Consumes: `CONF_WEATHER_ENTITY`, `check_input` XOR rules
- Produces: one-screen form with optional weather picker; weather unique_id `weather-{id}`

- [ ] **Step 1: Write failing config-flow tests**

Keep `test_successful_config_flow` submitting today's T+H fields (no weather key).

Add:

```python
from custom_components.thermal_comfort.const import CONF_WEATHER_ENTITY
from .conftest import WEATHER_ENTITY_ID, async_set_weather

async def test_weather_config_flow(hass):
    async_set_weather(hass)
    result = await _flow_init(hass)
    assert result["type"] == FlowResultType.FORM
    result = await hass.config_entries.flow.async_configure(
        result["flow_id"],
        user_input={
            CONF_NAME: "Outside",
            CONF_WEATHER_ENTITY: WEATHER_ENTITY_ID,
        },
    )
    assert result["type"] == FlowResultType.CREATE_ENTRY
    assert result["title"] == "Outside"
    assert result["data"][CONF_WEATHER_ENTITY] == WEATHER_ENTITY_ID
    assert CONF_TEMPERATURE_SENSOR not in result["data"]


async def test_mixed_sources_error(hass, start_ha):
    async_set_weather(hass)
    result = await _flow_init(hass)
    result = await hass.config_entries.flow.async_configure(
        result["flow_id"],
        user_input={
            **USER_INPUT,
            CONF_WEATHER_ENTITY: WEATHER_ENTITY_ID,
        },
    )
    assert result["type"] == FlowResultType.FORM
    assert result["errors"]["base"] == "need_weather_or_sensors"
```

`test_mixed_sources_error` uses `USER_INPUT` which already has T+H. Parametrize with `DEFAULT_TEST_SENSORS` if `start_ha` is required so those sensors exist; weather is set in the test body.

- [ ] **Step 2: Run tests; they fail**

```bash
pytest tests/test_config_flow.py::test_weather_config_flow -v
```

Expected: FAIL (form still requires temperature and humidity).

- [ ] **Step 3: One-screen config flow**

In `build_schema`:

- Add optional weather entity selector: `{"entity": {"domain": "weather"}}`. Include the entry's current weather entity if it is missing from the state machine.
- Make T and H `vol.Optional` (keep defaults when the filtered lists are non-empty).
- Do **not** return `None` solely because T/H lists are empty if at least one `weather.*` state exists.
- Abort `no_sensors` only when there are no weather entities **and** no T/H sensors.

`check_input`:

```python
def check_input(hass: HomeAssistant, user_input: dict) -> dict:
    result = {}
    weather_id = user_input.get(CONF_WEATHER_ENTITY) or None
    temp_id = user_input.get(CONF_TEMPERATURE_SENSOR) or None
    hum_id = user_input.get(CONF_HUMIDITY_SENSOR) or None
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

On create: drop empty source keys from `data`. If weather: `unique_id = f"weather-{registry.unique_id or entity_id}"`. Else keep `{t}-{h}`.

Options flow: if the entry has `weather_entity`, show weather + advanced (no T/H). Else show today's T/H form (no weather field).

- [ ] **Step 4: English strings**

In `en.json` add:

- `weather_entity`: "Weather entity"
- `need_weather_or_sensors`: "Choose a weather entity or both a temperature and a humidity sensor"
- `weather_not_found`: "Weather entity not found"

Add the same keys under `config` and `options` error/data sections that already exist for T/H.

- [ ] **Step 5: Docs**

`documentation/yaml.md`: weather example from the spec; XOR rule; name still independent in this stage.

`documentation/config_flow.md`: optional Weather entity on the same form; weather-only installs do not need helper sensors; this does not move entities onto the weather device (later stage).

- [ ] **Step 6: Run all tests**

```bash
pytest tests/test_weather.py tests/test_config_flow.py tests/test_sensor.py tests/test_init.py -v
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add custom_components/thermal_comfort/config_flow.py custom_components/thermal_comfort/translations/en.json tests/test_config_flow.py documentation
git commit -m "$(cat <<'EOF'
feat: add weather entity to the Thermal Comfort config form

Users can select a weather entity instead of two helper sensors.
Comfort sensors still live on the existing virtual device.
EOF
)"
```

---

## Plan self-review

1. **Spec coverage:** Stage 1 only (weather **or** separate T/H on one device). Global/zone graph, device linking, forecasts, and advice are other files.
2. **Placeholders:** None for Stage 1. Formula extract, `forecast_next`, and `via_device` are intentionally absent.
3. **Types:** `CONF_WEATHER_ENTITY` is the only new stored key. Unique ID `weather-{id}`.
