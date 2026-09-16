# Weather Entity Input Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let users select a Home Assistant weather entity as the Thermal Comfort source so the existing sensors show current indices and a `forecast` attribute with projected values.

**Architecture:** Extract index math into pure functions in `calculations.py`. `DeviceThermalComfort` keeps HA lifecycle and gains a weather-entity path that reads current `temperature`/`humidity` attributes and calls `weather.get_forecasts`. Each `SensorThermalComfort` entity continues to represent one index; weather-mode sensors attach a `forecast` extra attribute. Sensor-mode config is unchanged.

**Tech Stack:** Home Assistant custom component (Python 3.11+, HA >= 2023.12.0), pytest-homeassistant-custom-component, voluptuous, config flow selectors.

**Spec:** `docs/superpowers/specs/2026-09-16-weather-entity-design.md`

## Global Constraints

- Home Assistant >= 2023.12.0
- Python >= 3.11.0
- Do not change calculation formulas
- Do not change unique IDs of existing sensor-mode entities
- Sensor-based and weather-based sources are mutually exclusive on one device
- Config entry `VERSION` stays 2; weather keys are additive
- English translations only (`en.json`); other locales fall back to English
- Follow Home Assistant development guidelines already used by this repo
- Existing YAML `temperature_sensor` + `humidity_sensor` remains valid

## File map

- Create: `custom_components/thermal_comfort/calculations.py` — pure index functions + `calculate_sensor` dispatcher
- Create: `tests/test_calculations.py` — formula contract without Home Assistant
- Create: `tests/test_weather.py` — weather current values, forecasts, YAML exclusive source
- Modify: `custom_components/thermal_comfort/const.py` — `CONF_WEATHER_ENTITY`, `CONF_FORECAST_TYPE`, forecast-type constants, feature flags
- Modify: `custom_components/thermal_comfort/sensor.py` — schema, weather listeners, forecast attributes, thin wrappers over `calculations.py`
- Modify: `custom_components/thermal_comfort/__init__.py` — copy weather keys into `hass.data`
- Modify: `custom_components/thermal_comfort/config_flow.py` — `user` → `sensors` | `weather` steps; weather options flow
- Modify: `custom_components/thermal_comfort/translations/en.json` — new flow strings
- Modify: `tests/test_config_flow.py` — two-step sensor flow + weather flow tests
- Modify: `tests/conftest.py` / `tests/const.py` — weather helpers and sample input
- Modify: `documentation/yaml.md`, `documentation/config_flow.md`

---

### Task 1: Extract index calculations

**Files:**
- Create: `custom_components/thermal_comfort/calculations.py`
- Create: `tests/test_calculations.py`
- Modify: `custom_components/thermal_comfort/sensor.py` (index methods become wrappers)

**Interfaces:**
- Consumes: `SensorType` and perception enums / `ATTR_*` constants from `sensor.py`
- Produces: `calculate_<sensor_type>(temperature: float, humidity: float)` functions; `calculate_sensor(sensor_type: SensorType, temperature: float, humidity: float) -> float | tuple`

- [ ] **Step 1: Write the failing unit tests**

Create `tests/test_calculations.py`. These values are the existing contract from `tests/test_sensor.py` at 25 °C / 50 % RH.

```python
"""Unit tests for thermal comfort calculations."""
import math

import pytest

from custom_components.thermal_comfort.calculations import (
    calculate_absolute_humidity,
    calculate_dew_point,
    calculate_dew_point_perception,
    calculate_heat_index,
    calculate_humidex,
    calculate_sensor,
)
from custom_components.thermal_comfort.sensor import (
    DewPointPerception,
    SensorType,
)


def test_absolute_humidity_25c_50rh():
    assert calculate_absolute_humidity(25.0, 50.0) == pytest.approx(11.5128065738593)


def test_heat_index_25c_50rh():
    assert calculate_heat_index(25.0, 50.0) == pytest.approx(24.8611111111111)


def test_humidex_25c_50rh():
    assert calculate_humidex(25.0, 50.0) == pytest.approx(28.2925656121491)


def test_dew_point_perception_returns_tuple():
    perception, extras = calculate_dew_point_perception(25.0, 50.0)
    assert perception == DewPointPerception.COMFORTABLE
    assert "dew_point" in extras
    assert extras["dew_point"] == pytest.approx(calculate_dew_point(25.0, 50.0))


def test_calculate_sensor_dispatches_numeric_and_perception():
    abs_h = calculate_sensor(SensorType.ABSOLUTE_HUMIDITY, 25.0, 50.0)
    assert abs_h == pytest.approx(11.5128065738593)
    perception, extras = calculate_sensor(
        SensorType.DEW_POINT_PERCEPTION, 25.0, 50.0
    )
    assert perception == DewPointPerception.COMFORTABLE
    assert math.isfinite(extras["dew_point"])
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/test_calculations.py -v`

Expected: FAIL with `ModuleNotFoundError: No module named 'custom_components.thermal_comfort.calculations'`

- [ ] **Step 3: Add `calculations.py` with the existing formulas**

Create `custom_components/thermal_comfort/calculations.py`. Copy formulas from `DeviceThermalComfort` in `sensor.py` unchanged. Replace `self._temperature` with `temperature`, `self._humidity` with `humidity`, and `await self.<method>()` with the matching `calculate_*` call.

```python
"""Pure thermal-comfort index calculations.

Temperature is Celsius. Humidity is relative humidity in percent.
Formulas must stay identical to DeviceThermalComfort in sensor.py.
"""
from __future__ import annotations

import math
from collections.abc import Callable

from homeassistant.const import UnitOfTemperature
from homeassistant.util.unit_conversion import TemperatureConverter

from .sensor import (
    ATTR_DEW_POINT,
    ATTR_FROST_POINT,
    ATTR_HUMIDEX,
    ATTR_RELATIVE_STRAIN_INDEX,
    ATTR_SUMMER_SCHARLAU_INDEX,
    ATTR_SUMMER_SIMMER_INDEX,
    ATTR_THOMS_DISCOMFORT_INDEX,
    ATTR_WINTER_SCHARLAU_INDEX,
    DewPointPerception,
    FrostRisk,
    HumidexPerception,
    RelativeStrainPerception,
    ScharlauPerception,
    SensorType,
    SummerSimmerPerception,
    ThomsDiscomfortPerception,
)


def calculate_dew_point(temperature: float, humidity: float) -> float:
    """Dew Point <http://wahiduddin.net/calc/density_algorithms.htm>."""
    A0 = 373.15 / (273.15 + temperature)
    SUM = -7.90298 * (A0 - 1)
    SUM += 5.02808 * math.log(A0, 10)
    SUM += -1.3816e-7 * (pow(10, (11.344 * (1 - 1 / A0))) - 1)
    SUM += 8.1328e-3 * (pow(10, (-3.49149 * (A0 - 1))) - 1)
    SUM += math.log(1013.246, 10)
    VP = pow(10, SUM - 3) * humidity
    Td = math.log(VP / 0.61078)
    Td = (241.88 * Td) / (17.558 - Td)
    return Td


def calculate_heat_index(temperature: float, humidity: float) -> float:
    """Heat Index <http://www.wpc.ncep.noaa.gov/html/heatindex_equation.shtml>."""
    fahrenheit = TemperatureConverter.convert(
        temperature, UnitOfTemperature.CELSIUS, UnitOfTemperature.FAHRENHEIT
    )
    hi = 0.5 * (
        fahrenheit + 61.0 + ((fahrenheit - 68.0) * 1.2) + (humidity * 0.094)
    )

    if hi > 79:
        hi = -42.379 + 2.04901523 * fahrenheit
        hi = hi + 10.14333127 * humidity
        hi = hi + -0.22475541 * fahrenheit * humidity
        hi = hi + -0.00683783 * pow(fahrenheit, 2)
        hi = hi + -0.05481717 * pow(humidity, 2)
        hi = hi + 0.00122874 * pow(fahrenheit, 2) * humidity
        hi = hi + 0.00085282 * fahrenheit * pow(humidity, 2)
        hi = hi + -0.00000199 * pow(fahrenheit, 2) * pow(humidity, 2)

    if humidity < 13 and fahrenheit >= 80 and fahrenheit <= 112:
        hi = hi - ((13 - humidity) * 0.25) * math.sqrt(
            (17 - abs(fahrenheit - 95)) * 0.05882
        )
    elif humidity > 85 and fahrenheit >= 80 and fahrenheit <= 87:
        hi = hi + ((humidity - 85) * 0.1) * ((87 - fahrenheit) * 0.2)

    return TemperatureConverter.convert(
        hi, UnitOfTemperature.FAHRENHEIT, UnitOfTemperature.CELSIUS
    )


def calculate_humidex(temperature: float, humidity: float) -> float:
    """<https://simple.wikipedia.org/wiki/Humidex#Humidex_formula>."""
    dewpoint = calculate_dew_point(temperature, humidity)
    e = 6.11 * math.exp(5417.7530 * ((1 / 273.16) - (1 / (dewpoint + 273.15))))
    h = (0.5555) * (e - 10.0)
    return temperature + h


def calculate_humidex_perception(
    temperature: float, humidity: float
) -> tuple[HumidexPerception, dict]:
    """<https://simple.wikipedia.org/wiki/Humidex#Humidex_formula>."""
    humidex = calculate_humidex(temperature, humidity)
    if humidex > 54:
        perception = HumidexPerception.HEAT_STROKE
    elif humidex >= 45:
        perception = HumidexPerception.DANGEROUS_DISCOMFORT
    elif humidex >= 40:
        perception = HumidexPerception.GREAT_DISCOMFORT
    elif humidex >= 35:
        perception = HumidexPerception.EVIDENT_DISCOMFORT
    elif humidex >= 30:
        perception = HumidexPerception.NOTICABLE_DISCOMFORT
    else:
        perception = HumidexPerception.COMFORTABLE

    return perception, {ATTR_HUMIDEX: humidex}


def calculate_dew_point_perception(
    temperature: float, humidity: float
) -> tuple[DewPointPerception, dict]:
    """Dew Point <https://en.wikipedia.org/wiki/Dew_point>."""
    dewpoint = calculate_dew_point(temperature, humidity)
    if dewpoint < 10:
        perception = DewPointPerception.DRY
    elif dewpoint < 13:
        perception = DewPointPerception.VERY_COMFORTABLE
    elif dewpoint < 16:
        perception = DewPointPerception.COMFORTABLE
    elif dewpoint < 18:
        perception = DewPointPerception.OK_BUT_HUMID
    elif dewpoint < 21:
        perception = DewPointPerception.SOMEWHAT_UNCOMFORTABLE
    elif dewpoint < 24:
        perception = DewPointPerception.QUITE_UNCOMFORTABLE
    elif dewpoint < 26:
        perception = DewPointPerception.EXTREMELY_UNCOMFORTABLE
    else:
        perception = DewPointPerception.SEVERELY_HIGH

    return perception, {ATTR_DEW_POINT: dewpoint}


def calculate_absolute_humidity(temperature: float, humidity: float) -> float:
    """Absolute Humidity <https://carnotcycle.wordpress.com/2012/08/04/how-to-convert-relative-humidity-to-absolute-humidity/>."""
    abs_temperature = temperature + 273.15
    abs_humidity = 6.112
    abs_humidity *= math.exp((17.67 * temperature) / (243.5 + temperature))
    abs_humidity *= humidity
    abs_humidity *= 2.1674
    abs_humidity /= abs_temperature
    return abs_humidity


def calculate_frost_point(temperature: float, humidity: float) -> float:
    """Frost Point <https://pon.fr/dzvents-alerte-givre-et-calcul-humidite-absolue/>."""
    dewpoint = calculate_dew_point(temperature, humidity)
    T = temperature + 273.15
    Td = dewpoint + 273.15
    return (
        Td + (2671.02 / ((2954.61 / T) + 2.193665 * math.log(T) - 13.3448)) - T
    ) - 273.15


def calculate_frost_risk(
    temperature: float, humidity: float
) -> tuple[FrostRisk, dict]:
    """Frost Risk Level."""
    thresholdAbsHumidity = 2.8
    absolutehumidity = calculate_absolute_humidity(temperature, humidity)
    frostpoint = calculate_frost_point(temperature, humidity)
    if temperature <= 1 and frostpoint <= 0:
        if absolutehumidity <= thresholdAbsHumidity:
            frost_risk = FrostRisk.LOW
        else:
            frost_risk = FrostRisk.HIGH
    elif (
        temperature <= 4
        and frostpoint <= 0.5
        and absolutehumidity > thresholdAbsHumidity
    ):
        frost_risk = FrostRisk.MEDIUM
    else:
        frost_risk = FrostRisk.NONE

    return frost_risk, {ATTR_FROST_POINT: frostpoint}


def calculate_relative_strain_perception(
    temperature: float, humidity: float
) -> tuple[RelativeStrainPerception, dict]:
    """Relative strain perception."""
    vp = 6.112 * pow(10, 7.5 * temperature / (237.7 + temperature))
    e = humidity * vp / 100
    rsi = round((temperature - 21) / (58 - e), 2)

    if temperature < 26 or temperature > 35:
        perception = RelativeStrainPerception.OUTSIDE_CALCULABLE_RANGE
    elif rsi >= 0.45:
        perception = RelativeStrainPerception.EXTREME_DISCOMFORT
    elif rsi >= 0.35:
        perception = RelativeStrainPerception.SIGNIFICANT_DISCOMFORT
    elif rsi >= 0.25:
        perception = RelativeStrainPerception.DISCOMFORT
    elif rsi >= 0.15:
        perception = RelativeStrainPerception.SLIGHT_DISCOMFORT
    else:
        perception = RelativeStrainPerception.COMFORTABLE

    return perception, {ATTR_RELATIVE_STRAIN_INDEX: rsi}


def calculate_summer_scharlau_perception(
    temperature: float, humidity: float
) -> tuple[ScharlauPerception, dict]:
    """<https://revistadechimie.ro/pdf/16%20RUSANESCU%204%2019.pdf>."""
    tc = -17.089 * math.log(humidity) + 94.979
    ise = tc - temperature

    if temperature < 17 or temperature > 39 or humidity < 30:
        perception = ScharlauPerception.OUTSIDE_CALCULABLE_RANGE
    elif ise <= -3:
        perception = ScharlauPerception.HIGHLY_UNCOMFORTABLE
    elif ise <= -1:
        perception = ScharlauPerception.MODERATELY_UNCOMFORTABLE
    elif ise < 0:
        perception = ScharlauPerception.SLIGHTLY_UNCOMFORTABLE
    else:
        perception = ScharlauPerception.COMFORTABLE

    return perception, {ATTR_SUMMER_SCHARLAU_INDEX: round(ise, 2)}


def calculate_winter_scharlau_perception(
    temperature: float, humidity: float
) -> tuple[ScharlauPerception, dict]:
    """<https://revistadechimie.ro/pdf/16%20RUSANESCU%204%2019.pdf>."""
    tc = -0.0003 * humidity**2 + 0.1497 * humidity - 7.7133
    ish = temperature - tc
    if temperature < -5 or temperature > 6 or humidity < 40:
        perception = ScharlauPerception.OUTSIDE_CALCULABLE_RANGE
    elif ish <= -3:
        perception = ScharlauPerception.HIGHLY_UNCOMFORTABLE
    elif ish <= -1:
        perception = ScharlauPerception.MODERATELY_UNCOMFORTABLE
    elif ish < 0:
        perception = ScharlauPerception.SLIGHTLY_UNCOMFORTABLE
    else:
        perception = ScharlauPerception.COMFORTABLE

    return perception, {ATTR_WINTER_SCHARLAU_INDEX: round(ish, 2)}


def calculate_summer_simmer_index(temperature: float, humidity: float) -> float:
    """<https://www.vcalc.com/wiki/rklarsen/Summer+Simmer+Index>."""
    fahrenheit = TemperatureConverter.convert(
        temperature, UnitOfTemperature.CELSIUS, UnitOfTemperature.FAHRENHEIT
    )

    si = (
        1.98
        * (fahrenheit - (0.55 - (0.0055 * humidity)) * (fahrenheit - 58.0))
        - 56.83
    )

    if fahrenheit < 58:
        si = fahrenheit

    return TemperatureConverter.convert(
        si, UnitOfTemperature.FAHRENHEIT, UnitOfTemperature.CELSIUS
    )


def calculate_summer_simmer_perception(
    temperature: float, humidity: float
) -> tuple[SummerSimmerPerception, dict]:
    """<http://summersimmer.com/default.asp>."""
    si = calculate_summer_simmer_index(temperature, humidity)
    if si < 21.1:
        summer_simmer_perception = SummerSimmerPerception.COOL
    elif si < 25.0:
        summer_simmer_perception = SummerSimmerPerception.SLIGHTLY_COOL
    elif si < 28.3:
        summer_simmer_perception = SummerSimmerPerception.COMFORTABLE
    elif si < 32.8:
        summer_simmer_perception = SummerSimmerPerception.SLIGHTLY_WARM
    elif si < 37.8:
        summer_simmer_perception = SummerSimmerPerception.INCREASING_DISCOMFORT
    elif si < 44.4:
        summer_simmer_perception = SummerSimmerPerception.EXTREMELY_WARM
    elif si < 51.7:
        summer_simmer_perception = SummerSimmerPerception.DANGER_OF_HEATSTROKE
    elif si < 65.6:
        summer_simmer_perception = (
            SummerSimmerPerception.EXTREME_DANGER_OF_HEATSTROKE
        )
    else:
        summer_simmer_perception = (
            SummerSimmerPerception.CIRCULATORY_COLLAPSE_IMMINENT
        )

    return summer_simmer_perception, {ATTR_SUMMER_SIMMER_INDEX: si}


def calculate_moist_air_enthalpy(temperature: float, humidity: float) -> float:
    """Calculate the enthalpy of moist air."""
    patm = 101325
    c_to_k = 273.15

    c1 = -5.6745359e03
    c2 = 6.3925247e00
    c3 = -9.6778430e-03
    c4 = 6.2215701e-07
    c5 = 2.0747825e-09
    c6 = -9.4840240e-13
    c7 = 4.1635019e00
    c8 = -5.8002206e03
    c9 = 1.3914993e00
    c10 = -4.8640239e-02
    c11 = 4.1764768e-05
    c12 = -1.4452093e-08
    c13 = 6.5459673e00

    T = temperature + c_to_k

    p_ws = (
        math.exp(c1 / T + c2 + c3 * T + c4 * T**2 + c5 * T**3 + c6 * T**4 + c7 * math.log(T))
        if T < c_to_k  # noqa: SIM300
        else math.exp(c8 / T + c9 + c10 * T + c11 * T**2 + c12 * T**3 + c13 * math.log(T))
    )

    p_w = humidity / 100 * p_ws
    W = 0.621945 * p_w / (patm - p_w)
    return 1.006 * temperature + W * (2501 + 1.86 * temperature)


def calculate_thoms_discomfort_perception(
    temperature: float, humidity: float
) -> tuple[ThomsDiscomfortPerception, dict]:
    """Calculate Thom's discomfort index and perception."""
    tw = (
        temperature
        * math.atan(0.151977 * pow(humidity + 8.313659, 1 / 2))
        + math.atan(temperature + humidity)
        - math.atan(humidity - 1.676331)
        + pow(0.00391838 * humidity, 3 / 2)
        * math.atan(0.023101 * humidity)
        - 4.686035
    )
    tdi = 0.5 * tw + 0.5 * temperature

    if tdi >= 32:
        perception = ThomsDiscomfortPerception.DANGEROUS
    elif tdi >= 29:
        perception = ThomsDiscomfortPerception.EVERYONE
    elif tdi >= 27:
        perception = ThomsDiscomfortPerception.MOST
    elif tdi >= 24:
        perception = ThomsDiscomfortPerception.MORE_THAN_HALF
    elif tdi >= 21:
        perception = ThomsDiscomfortPerception.LESS_THAN_HALF
    else:
        perception = ThomsDiscomfortPerception.NO_DISCOMFORT

    return perception, {ATTR_THOMS_DISCOMFORT_INDEX: round(tdi, 2)}


SENSOR_CALCULATORS: dict[SensorType, Callable[[float, float], object]] = {
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


def calculate_sensor(
    sensor_type: SensorType, temperature: float, humidity: float
) -> object:
    """Dispatch to the calculator for ``sensor_type``."""
    return SENSOR_CALCULATORS[sensor_type](temperature, humidity)
```

Circular import: `calculations.py` imports enums/ATTR from `sensor.py`, and `sensor.py` will import functions from `calculations.py`. Avoid that by moving the perception enums and `ATTR_*` used by calculations into `const.py`, **or** keep enums in `sensor.py` and import them inside each `calculate_*` only if a cycle appears. Preferred: import `calculations` at the bottom of the wrapper methods' module level **after** enums are defined. `sensor.py` already defines enums before `DeviceThermalComfort`, so `from .calculations import ...` at the top of `sensor.py` will cycle.

**Fix the cycle:** put the `from .calculations import ...` import inside `DeviceThermalComfort` methods, **or** move `SensorType`, perception enums, and calculation `ATTR_*` constants into `const.py` first. Use a lazy import in `sensor.py`:

```python
# at the end of sensor.py, after SensorType and perception enums exist, this still cycles
# if calculations imports sensor at module load.

# Instead, in calculations.py import ATTR and enums from a new thin module.
```

Do this: leave enums in `sensor.py`. In `calculations.py` **do not** import `sensor` at module top. Import inside `calculate_sensor` and each function that needs an enum... that is messy.

Cleaner: add at the top of `calculations.py` only `TYPE_CHECKING` imports, and pass perception classes as values already defined. Simplest cycle break used in this codebase style:

In `calculations.py`, import from `sensor` **inside** `calculate_sensor` and keep perception returns as strings? That would change types.

**Chosen cycle break:** move these symbols from `sensor.py` into `const.py` is too large. Use late import in `sensor.py`:

```python
# sensor.py — do NOT import calculations at module top.

# DeviceThermalComfort.dew_point:
async def dew_point(self) -> float:
    from .calculations import calculate_dew_point
    return calculate_dew_point(self._temperature, self._humidity)
```

That is 16 late imports. Better: import once in `DeviceThermalComfort.__init__` after the class exists, still cycles.

**Actual cycle break:** in `calculations.py`, duplicate is not allowed. Import enums from `sensor` at function level in `SENSOR_CALCULATORS` builder:

```python
def _calculators():
    from .sensor import SensorType
    return { SensorType.DEW_POINT: calculate_dew_point, ... }

def calculate_sensor(sensor_type, temperature, humidity):
    return _calculators()[sensor_type](temperature, humidity)
```

Still need enum classes inside calculate_dew_point_perception at runtime — import them at the start of each perception function:

```python
def calculate_dew_point_perception(temperature: float, humidity: float):
    from .sensor import ATTR_DEW_POINT, DewPointPerception
    ...
```

Do that for every function that needs `sensor.py` symbols. `tests/test_calculations.py` can still `from custom_components.thermal_comfort.calculations import calculate_dew_point`.

- [ ] **Step 4: Replace DeviceThermalComfort method bodies with wrappers**

In `custom_components/thermal_comfort/sensor.py`, keep the method names, docstrings, and `@compute_once_lock` decorators. Replace the formula body with a call. Example for dew point and humidex perception:

```python
    @compute_once_lock(SensorType.DEW_POINT)
    async def dew_point(self) -> float:
        """Dew Point <http://wahiduddin.net/calc/density_algorithms.htm>."""
        from .calculations import calculate_dew_point

        return calculate_dew_point(self._temperature, self._humidity)

    @compute_once_lock(SensorType.HUMIDEX_PERCEPTION)
    async def humidex_perception(self) -> (HumidexPerception, dict):
        """<https://simple.wikipedia.org/wiki/Humidex#Humidex_formula>."""
        from .calculations import calculate_humidex_perception

        return calculate_humidex_perception(self._temperature, self._humidity)
```

Repeat for: `heat_index`, `humidex`, `dew_point_perception`, `absolute_humidity`, `frost_point`, `frost_risk`, `relative_strain_perception`, `summer_scharlau_perception`, `winter_scharlau_perception`, `summer_simmer_index`, `summer_simmer_perception`, `moist_air_enthalpy`, `thoms_discomfort_perception`.

- [ ] **Step 5: Run unit tests and existing sensor tests**

Run:

```bash
pytest tests/test_calculations.py tests/test_sensor.py -v
```

Expected: PASS. Existing numeric strings in `test_sensor.py` must be unchanged.

- [ ] **Step 6: Commit**

```bash
git add custom_components/thermal_comfort/calculations.py custom_components/thermal_comfort/sensor.py tests/test_calculations.py
git commit -m "$(cat <<'EOF'
refactor: extract thermal comfort calculations for reuse

Move index formulas into pure functions so current and forecast
paths can share one implementation without changing results.
EOF
)"
```

---

### Task 2: Weather current values and YAML source

**Files:**
- Modify: `custom_components/thermal_comfort/const.py`
- Modify: `custom_components/thermal_comfort/sensor.py` (`SENSOR_SCHEMA`, `DeviceThermalComfort`, setup)
- Modify: `custom_components/thermal_comfort/__init__.py`
- Modify: `tests/conftest.py`
- Create: `tests/test_weather.py`

**Interfaces:**
- Consumes: `calculate_*` from Task 1; existing `DeviceThermalComfort.async_shutdown` / availability
- Produces: `CONF_WEATHER_ENTITY`, `CONF_FORECAST_TYPE`, `FORECAST_TYPE_AUTO`, `FORECAST_TYPES`; `DeviceThermalComfort(..., weather_entity: str | None = None, forecast_type: str = FORECAST_TYPE_AUTO)`; YAML exclusive source validation

- [ ] **Step 1: Write failing tests for weather current values and YAML exclusivity**

Add helpers to `tests/conftest.py`:

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
    """Create a weather entity with current attributes."""
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

Create `tests/test_weather.py`:

```python
"""Tests for weather-entity input."""
import pytest
from pytest_homeassistant_custom_component.common import MockConfigEntry
from voluptuous.error import Invalid

from custom_components.thermal_comfort.const import (
    CONF_FORECAST_TYPE,
    CONF_WEATHER_ENTITY,
    DOMAIN,
    FORECAST_TYPE_AUTO,
)
from custom_components.thermal_comfort.sensor import (
    SENSOR_SCHEMA,
    SensorType,
)
from homeassistant.components.sensor import DOMAIN as PLATFORM_DOMAIN
from homeassistant.const import ATTR_TEMPERATURE, UnitOfTemperature
from homeassistant.core import HomeAssistant

from .conftest import WEATHER_ENTITY_ID, async_set_weather
from .test_sensor import LEN_DEFAULT_SENSORS, get_sensor

WEATHER_YAML = {
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
}


@pytest.fixture
async def start_ha_weather(hass, domains, config):
    """Set up thermal_comfort from a weather entity."""
    async_set_weather(hass)
    await hass.async_block_till_done()
    from pytest_homeassistant_custom_component.common import assert_setup_component
    from homeassistant.setup import async_setup_component

    for domain, count in domains:
        with assert_setup_component(count, domain):
            assert await async_setup_component(hass, domain, config)
        await hass.async_block_till_done()
    await hass.async_start()
    await hass.async_block_till_done()


@pytest.mark.parametrize(*WEATHER_YAML)
async def test_weather_current_values(hass: HomeAssistant, start_ha_weather) -> None:
    """Weather attributes drive the same current sensors as source sensors."""
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).state == "11.5128065738593"
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).attributes[ATTR_TEMPERATURE] == 25.0
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).attributes["humidity"] == 50.0


@pytest.mark.parametrize(*WEATHER_YAML)
async def test_weather_fahrenheit_current_values(
    hass: HomeAssistant, start_ha_weather
) -> None:
    """Convert weather temperature_unit before calculating."""
    async_set_weather(hass, temperature=77.0, humidity=50.0, unit=UnitOfTemperature.FAHRENHEIT)
    await hass.async_block_till_done()
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).state == "11.5128065738593"
    assert get_sensor(hass, SensorType.ABSOLUTE_HUMIDITY).attributes[ATTR_TEMPERATURE] == 77.0


@pytest.mark.parametrize(*WEATHER_YAML)
async def test_weather_unavailable_makes_sensors_unavailable(
    hass: HomeAssistant, start_ha_weather
) -> None:
    """Unavailable weather entity marks calculated sensors unavailable."""
    hass.states.async_set(WEATHER_ENTITY_ID, "unavailable")
    await hass.async_block_till_done()
    from homeassistant.const import STATE_UNAVAILABLE

    assert get_sensor(hass, SensorType.DEW_POINT).state == STATE_UNAVAILABLE


def test_yaml_rejects_mixed_sources() -> None:
    """weather_entity cannot be combined with temperature/humidity sensors."""
    with pytest.raises(Invalid):
        SENSOR_SCHEMA(
            {
                "name": "Mixed",
                "weather_entity": WEATHER_ENTITY_ID,
                "temperature_sensor": "sensor.temp",
                "humidity_sensor": "sensor.hum",
                "unique_id": "mixed",
            }
        )


def test_yaml_requires_a_source() -> None:
    """A device must have weather_entity or both sensors."""
    with pytest.raises(Invalid):
        SENSOR_SCHEMA({"name": "None", "unique_id": "none"})
```

The `start_ha_weather` fixture duplicates `start_ha` except it calls `async_set_weather` instead of `async_set_source_sensors`. Prefer extracting a shared setup helper in `conftest.py` if the duplication is noisy; behavior must match `start_ha`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/test_weather.py -v`

Expected: FAIL (`CONF_WEATHER_ENTITY` missing and/or YAML still requires temperature/humidity sensors).

- [ ] **Step 3: Add constants**

In `custom_components/thermal_comfort/const.py` append:

```python
CONF_WEATHER_ENTITY = "weather_entity"
CONF_FORECAST_TYPE = "forecast_type"
CONF_INPUT_SOURCE = "input_source"

INPUT_SOURCE_SENSORS = "sensors"
INPUT_SOURCE_WEATHER = "weather"

FORECAST_TYPE_AUTO = "auto"
FORECAST_TYPE_HOURLY = "hourly"
FORECAST_TYPE_DAILY = "daily"
FORECAST_TYPE_TWICE_DAILY = "twice_daily"
FORECAST_TYPES = [
    FORECAST_TYPE_AUTO,
    FORECAST_TYPE_HOURLY,
    FORECAST_TYPE_DAILY,
    FORECAST_TYPE_TWICE_DAILY,
]

# WeatherEntityFeature flags (avoid importing the weather component).
WEATHER_FORECAST_DAILY = 1
WEATHER_FORECAST_HOURLY = 2
WEATHER_FORECAST_TWICE_DAILY = 4

ATTR_FORECAST = "forecast"
ATTR_FORECAST_TYPE = "forecast_type"
```

- [ ] **Step 4: Relax YAML schema to exclusive sources**

In `sensor.py`, import the new constants. Replace `SENSOR_SCHEMA` with:

```python
def _validate_sensor_source(config: dict) -> dict:
    """Require weather_entity XOR both temperature and humidity sensors."""
    has_weather = CONF_WEATHER_ENTITY in config
    has_temperature = CONF_TEMPERATURE_SENSOR in config
    has_humidity = CONF_HUMIDITY_SENSOR in config
    if has_weather:
        extra = [key for key in (CONF_TEMPERATURE_SENSOR, CONF_HUMIDITY_SENSOR) if key in config]
        if extra:
            raise vol.Invalid(
                "weather_entity cannot be combined with temperature_sensor or humidity_sensor"
            )
        return config
    if has_temperature and has_humidity:
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
            vol.Optional(CONF_FORECAST_TYPE, default=FORECAST_TYPE_AUTO): vol.In(
                FORECAST_TYPES
            ),
            vol.Optional(CONF_ICON_TEMPLATE): cv.template,
            vol.Optional(CONF_ENTITY_PICTURE_TEMPLATE): cv.template,
            vol.Required(CONF_UNIQUE_ID): cv.string,
        }
    ).extend(SENSOR_OPTIONS_SCHEMA.schema),
    _validate_sensor_source,
)
```

- [ ] **Step 5: Teach DeviceThermalComfort to read a weather entity**

Change the constructor signature:

```python
    def __init__(
        self,
        hass: HomeAssistant,
        name: str,
        unique_id: str,
        temperature_entity: str | None,
        humidity_entity: str | None,
        should_poll: bool,
        scan_interval: timedelta,
        weather_entity: str | None = None,
        forecast_type: str = FORECAST_TYPE_AUTO,
    ):
```

Store `self._weather_entity = weather_entity` and `self._forecast_type = forecast_type`. Keep `self._forecasts: dict[SensorType, list[dict]] = {}` (empty until Task 4) and `self._resolved_forecast_type: str | None = None`.

If `weather_entity` is set, do **not** subscribe to temperature/humidity sensors. Subscribe once:

```python
        if self._weather_entity:
            self._unsub_callbacks.append(
                async_track_state_change_event(
                    self.hass, self._weather_entity, self.weather_state_listener
                )
            )
            hass.async_create_task(
                self._new_weather_state(hass.states.get(self._weather_entity))
            )
        else:
            self._unsub_callbacks.append(
                async_track_state_change_event(
                    self.hass, self._temperature_entity, self.temperature_state_listener
                )
            )
            self._unsub_callbacks.append(
                async_track_state_change_event(
                    self.hass, self._humidity_entity, self.humidity_state_listener
                )
            )
            hass.async_create_task(
                self._new_temperature_state(hass.states.get(temperature_entity))
            )
            hass.async_create_task(
                self._new_humidity_state(hass.states.get(humidity_entity))
            )
```

Add listeners. Temperature conversion and range checks must match `_new_temperature_state` / `_new_humidity_state`:

```python
    async def weather_state_listener(self, event) -> None:
        """Handle weather entity state changes."""
        if self._shutdown:
            return
        await self._new_weather_state(event.data.get("new_state"))

    async def _new_weather_state(self, state) -> None:
        """Read current temperature and humidity attributes from a weather entity."""
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
        raw_temperature = state.attributes.get("temperature")
        raw_humidity = state.attributes.get("humidity")

        temperature_ok = False
        humidity_ok = False
        try:
            temp = util.convert(raw_temperature, float)
            temperature = TemperatureConverter.convert(
                temp, unit, UnitOfTemperature.CELSIUS
            )
        except (TypeError, ValueError):
            temperature = None
        else:
            if temperature is not None and -89.2 <= temperature <= 56.7:
                self.extra_state_attributes[ATTR_TEMPERATURE] = temp
                self._temperature = temperature
                temperature_ok = True

        try:
            humidity = float(raw_humidity)
        except (TypeError, ValueError):
            humidity = None
        else:
            if 0 < humidity <= 100:
                self._humidity = humidity
                self.extra_state_attributes[ATTR_HUMIDITY] = humidity
                humidity_ok = True

        if not temperature_ok:
            self._temperature = None
            self.extra_state_attributes.pop(ATTR_TEMPERATURE, None)
        if not humidity_ok:
            self._humidity = None
            self.extra_state_attributes.pop(ATTR_HUMIDITY, None)

        if temperature_ok and humidity_ok:
            await self.async_update()
            return
        _LOGGER.info(
            "Weather entity %s is missing a valid temperature or humidity. "
            "Calculated states are unavailable. state=%s",
            self._weather_entity,
            state,
        )
        await self.async_update_sensors(True)
```

Pass `weather_entity` / `forecast_type` from `async_setup_platform` and `async_setup_entry`:

```python
            weather_entity=device_config.get(CONF_WEATHER_ENTITY),
            forecast_type=device_config.get(CONF_FORECAST_TYPE, FORECAST_TYPE_AUTO),
```

```python
        weather_entity=data.get(CONF_WEATHER_ENTITY),
        forecast_type=data.get(CONF_FORECAST_TYPE, FORECAST_TYPE_AUTO),
```

In `__init__.py` `async_setup_entry`, store the optional keys:

```python
        CONF_WEATHER_ENTITY: get_value(entry, CONF_WEATHER_ENTITY),
        CONF_FORECAST_TYPE: get_value(entry, CONF_FORECAST_TYPE, FORECAST_TYPE_AUTO),
```

Import those names from `const.py` (prefer `const.py` over duplicating in `sensor.py`). `sensor.py` may keep `CONF_TEMPERATURE_SENSOR` aliases as they are today; add `CONF_WEATHER_ENTITY = "weather_entity"` next to the existing sensor.py CONF aliases **or** import from const. Import from const to avoid a third copy.

- [ ] **Step 6: Run weather and existing tests**

Run:

```bash
pytest tests/test_weather.py tests/test_sensor.py tests/test_init.py -v
```

Expected: PASS, including `test_weather_current_values` and all previous sensor tests.

- [ ] **Step 7: Commit**

```bash
git add custom_components/thermal_comfort/const.py custom_components/thermal_comfort/sensor.py custom_components/thermal_comfort/__init__.py tests/conftest.py tests/test_weather.py
git commit -m "$(cat <<'EOF'
feat: accept a weather entity as the current T/RH source

Read temperature and humidity from weather attributes so outdoor
thermal comfort can be calculated without template sensors.
EOF
)"
```

---

### Task 3: Config flow weather source

**Files:**
- Modify: `custom_components/thermal_comfort/config_flow.py`
- Modify: `custom_components/thermal_comfort/translations/en.json`
- Modify: `tests/test_config_flow.py`
- Modify: `tests/const.py`

**Interfaces:**
- Consumes: `CONF_INPUT_SOURCE`, `INPUT_SOURCE_*`, `CONF_WEATHER_ENTITY`, `CONF_FORECAST_TYPE` from Task 2
- Produces: `async_step_user` (name + source) → `async_step_sensors` | `async_step_weather`; weather options flow; unique_id `weather-{unique_id}`

- [ ] **Step 1: Write failing config-flow tests**

Add to `tests/const.py`:

```python
from custom_components.thermal_comfort.const import (
    CONF_FORECAST_TYPE,
    CONF_WEATHER_ENTITY,
    FORECAST_TYPE_AUTO,
    INPUT_SOURCE_SENSORS,
    INPUT_SOURCE_WEATHER,
)

WEATHER_USER_INPUT = {
    CONF_NAME: "Outside",
    CONF_WEATHER_ENTITY: "weather.test",
    CONF_FORECAST_TYPE: FORECAST_TYPE_AUTO,
    CONF_POLL: False,
    CONF_CUSTOM_ICONS: False,
    CONF_SCAN_INTERVAL: 30,
}
```

Update existing sensor flow tests so the first configure call only submits name + source. Add helpers in `tests/test_config_flow.py`:

```python
from custom_components.thermal_comfort.const import (
    CONF_INPUT_SOURCE,
    CONF_WEATHER_ENTITY,
    INPUT_SOURCE_SENSORS,
    INPUT_SOURCE_WEATHER,
)
from .conftest import WEATHER_ENTITY_ID, async_set_weather
from .const import WEATHER_USER_INPUT

async def _configure_source(hass, result, source=INPUT_SOURCE_SENSORS, name=None):
    return await hass.config_entries.flow.async_configure(
        result["flow_id"],
        {
            CONF_NAME: name or ADVANCED_USER_INPUT[CONF_NAME],
            CONF_INPUT_SOURCE: source,
        },
    )
```

Change `test_successful_config_flow` to:

```python
@pytest.mark.parametrize(*DEFAULT_TEST_SENSORS)
async def test_successful_config_flow(hass, start_ha):
    result = await _flow_init(hass)
    assert result["type"] == FlowResultType.FORM
    assert result["step_id"] == "user"

    result = await _configure_source(hass, result)
    assert result["type"] == FlowResultType.FORM
    assert result["step_id"] == "sensors"

    result = await _flow_configure(hass, result)
    assert result["type"] == FlowResultType.CREATE_ENTRY
    assert result["title"] == ADVANCED_USER_INPUT[CONF_NAME]
    assert result["data"][CONF_TEMPERATURE_SENSOR] == ADVANCED_USER_INPUT[CONF_TEMPERATURE_SENSOR]
    assert CONF_INPUT_SOURCE not in result["data"]
```

`ADVANCED_USER_INPUT` still includes `CONF_NAME`; the sensors step should ignore a repeated name or omit name from its schema. Stored data must include `name` from step 1.

Add:

```python
async def test_weather_config_flow(hass):
    """Create an entry from a weather entity."""
    async_set_weather(hass)
    result = await _flow_init(hass)
    result = await _configure_source(
        hass, result, source=INPUT_SOURCE_WEATHER, name="Outside"
    )
    assert result["type"] == FlowResultType.FORM
    assert result["step_id"] == "weather"

    result = await hass.config_entries.flow.async_configure(
        result["flow_id"],
        user_input={
            CONF_WEATHER_ENTITY: WEATHER_ENTITY_ID,
            CONF_FORECAST_TYPE: FORECAST_TYPE_AUTO,
        },
    )
    assert result["type"] == FlowResultType.CREATE_ENTRY
    assert result["title"] == "Outside"
    assert result["data"][CONF_WEATHER_ENTITY] == WEATHER_ENTITY_ID
    assert CONF_TEMPERATURE_SENSOR not in result["data"]
    assert CONF_INPUT_SOURCE not in result["data"]


async def test_weather_options_flow(hass):
    """Weather entries keep weather fields in options."""
    async_set_weather(hass)
    entry = MockConfigEntry(
        domain=DOMAIN,
        data={
            CONF_NAME: "Outside",
            CONF_WEATHER_ENTITY: WEATHER_ENTITY_ID,
            CONF_FORECAST_TYPE: FORECAST_TYPE_AUTO,
        },
        entry_id="weather-test",
    )
    entry.add_to_hass(hass)
    result = await hass.config_entries.options.async_init(
        entry.entry_id, context={"show_advanced_options": True}
    )
    assert result["type"] == FlowResultType.FORM
    result = await hass.config_entries.options.async_configure(
        result["flow_id"],
        user_input={
            CONF_WEATHER_ENTITY: WEATHER_ENTITY_ID,
            CONF_FORECAST_TYPE: "hourly",
            CONF_POLL: False,
            CONF_CUSTOM_ICONS: False,
            CONF_SCAN_INTERVAL: 30,
        },
    )
    assert result["type"] == FlowResultType.CREATE_ENTRY
    assert entry.options[CONF_FORECAST_TYPE] == "hourly"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/test_config_flow.py -v`

Expected: FAIL — first step still shows temperature/humidity fields (`step_id == "user"` after configure with `input_source`).

- [ ] **Step 3: Implement the two-step flow**

In `ThermalComfortConfigFlow`:

```python
    def __init__(self) -> None:
        """Initialize the config flow."""
        self._user_input: dict = {}

    async def async_step_user(self, user_input=None):
        """Choose a name and input source."""
        if user_input is not None:
            self._user_input = dict(user_input)
            if user_input[CONF_INPUT_SOURCE] == INPUT_SOURCE_WEATHER:
                return await self.async_step_weather()
            return await self.async_step_sensors()

        return self.async_show_form(
            step_id="user",
            data_schema=vol.Schema(
                {
                    vol.Required(
                        CONF_NAME, default=DEFAULT_NAME
                    ): str,
                    vol.Required(
                        CONF_INPUT_SOURCE, default=INPUT_SOURCE_SENSORS
                    ): selector(
                        {
                            "select": {
                                "options": [
                                    INPUT_SOURCE_SENSORS,
                                    INPUT_SOURCE_WEATHER,
                                ],
                                "mode": "dropdown",
                                "translation_key": "input_source",
                            }
                        }
                    ),
                }
            ),
        )
```

Move today's sensor form into `async_step_sensors`. `build_schema(..., step="sensors")` must **omit** `CONF_NAME` (already collected). Unique-id logic stays here. Merge:

```python
                data = {**self._user_input, **user_input}
                data.pop(CONF_INPUT_SOURCE, None)
                return self.async_create_entry(title=data[CONF_NAME], data=data)
```

`async_step_weather`:

```python
    async def async_step_weather(self, user_input=None):
        """Configure a weather entity source."""
        errors = {}
        if user_input is not None:
            if not (errors := check_weather_input(self.hass, user_input)):
                registry = er.async_get(self.hass)
                weather = registry.async_get(user_input[CONF_WEATHER_ENTITY])
                unique_id = (
                    f"weather-{weather.unique_id}"
                    if weather is not None and weather.unique_id
                    else f"weather-{user_input[CONF_WEATHER_ENTITY]}"
                )
                await self.async_set_unique_id(unique_id)
                self._abort_if_unique_id_configured()
                data = {**self._user_input, **user_input}
                data.pop(CONF_INPUT_SOURCE, None)
                return self.async_create_entry(title=data[CONF_NAME], data=data)

        schema = build_weather_schema(
            config_entry=None,
            hass=self.hass,
            show_advanced=self.show_advanced_options,
            include_name=False,
        )
        if schema is None:
            return self.async_abort(reason="no_weather")
        return self.async_show_form(
            step_id="weather", data_schema=schema, errors=errors
        )
```

`check_weather_input`:

```python
def check_weather_input(hass: HomeAssistant, user_input: dict) -> dict:
    """Validate that the weather entity exists."""
    result = {}
    if hass.states.get(user_input[CONF_WEATHER_ENTITY]) is None:
        result[CONF_WEATHER_ENTITY] = "weather_not_found"
    return result
```

`build_weather_schema` lists `weather.*` states (`state.domain == "weather"`). Include the currently configured entity in options flow even if it is temporarily missing (same pattern as temperature/humidity in `build_schema`). Fields: `weather_entity` (required entity selector), `forecast_type` (select of `FORECAST_TYPES`, default `auto`), plus advanced poll/scan/custom_icons/`enabled_sensors` on the user step.

`ThermalComfortOptionsFlow.async_step_init`: if `get_value(self.config_entry, CONF_WEATHER_ENTITY)` is set, use `build_weather_schema` + `check_weather_input`; otherwise keep today's sensor schema + `check_input`. Do not offer `input_source` in options.

Keep `build_schema` working for sensor options. When `step == "sensors"`, do not include `CONF_NAME`.

- [ ] **Step 4: Add English strings**

In `custom_components/thermal_comfort/translations/en.json`:

```json
"config": {
  "abort": {
    "already_configured": "This combination of temperature and humidity sensors is already configured",
    "already_configured_weather": "This weather entity is already configured",
    "no_sensors": "No temperature or humidity sensors found. Try again in advanced mode.",
    "no_sensors_advanced": "No temperature or humidity sensors found.",
    "no_weather": "No weather entities found."
  },
  "error": {
    "temperature_not_found": "Temperature sensor not found",
    "humidity_not_found": "Humidity sensor not found",
    "weather_not_found": "Weather entity not found"
  },
  "step": {
    "user": {
      "title": "Thermal comfort settings",
      "data": {
        "name": "Name",
        "input_source": "Input source"
      }
    },
    "sensors": {
      "title": "Temperature and humidity sensors",
      "data": {
        "temperature_sensor": "Temperature sensor",
        "humidity_sensor": "Humidity sensor",
        "poll": "Enable Polling",
        "scan_interval": "Poll interval (seconds)",
        "custom_icons": "Use custom icons pack",
        "enabled_sensors": "Enabled sensors"
      }
    },
    "weather": {
      "title": "Weather entity",
      "data": {
        "weather_entity": "Weather entity",
        "forecast_type": "Forecast type",
        "poll": "Enable Polling",
        "scan_interval": "Poll interval (seconds)",
        "custom_icons": "Use custom icons pack",
        "enabled_sensors": "Enabled sensors"
      },
      "data_description": {
        "forecast_type": "Auto prefers hourly forecasts when the weather entity provides them."
      }
    }
  }
}
```

Also add under a top-level selector key if the HA version used by tests supports it:

```json
"selector": {
  "input_source": {
    "options": {
      "sensors": "Temperature and humidity sensors",
      "weather": "Weather entity"
    }
  },
  "forecast_type": {
    "options": {
      "auto": "Auto",
      "hourly": "Hourly",
      "daily": "Daily",
      "twice_daily": "Twice daily"
    }
  }
}
```

Mirror the new `error.weather_not_found` key under `options.error`. Add `options.step.init` labels for `weather_entity` and `forecast_type`. Keep existing sensor option labels.

If `already_configured` is used for both modes, keep the original abort reason for sensors and call `self._abort_if_unique_id_configured()` which uses `already_configured`. Override reason only if the flow API in this HA version allows it; otherwise keep `already_configured` for weather too and skip `already_configured_weather`.

- [ ] **Step 5: Run config-flow and weather tests**

Run:

```bash
pytest tests/test_config_flow.py tests/test_weather.py tests/test_sensor.py -v
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add custom_components/thermal_comfort/config_flow.py custom_components/thermal_comfort/translations/en.json tests/test_config_flow.py tests/const.py
git commit -m "$(cat <<'EOF'
feat: add config flow step for weather entity input

Let users pick sensors or a weather entity when creating a
Thermal Comfort device, without storing the source selector.
EOF
)"
```

---

### Task 4: Projected forecast values

**Files:**
- Modify: `custom_components/thermal_comfort/sensor.py` (`DeviceThermalComfort` forecast refresh, `SensorThermalComfort` attributes)
- Modify: `tests/test_weather.py`

**Interfaces:**
- Consumes: `calculate_sensor(sensor_type, temperature, humidity)` from Task 1; `weather.get_forecasts` service; `FORECAST_TYPE_*` and `WEATHER_FORECAST_*` from Task 2
- Produces: `DeviceThermalComfort._forecasts: dict[SensorType, list[dict]]`; sensor extra attribute `forecast`; unrecorded `forecast`; device extra `forecast_type` when resolved

- [ ] **Step 1: Write failing forecast tests**

Append to `tests/test_weather.py`:

```python
from custom_components.thermal_comfort.calculations import calculate_heat_index
from custom_components.thermal_comfort.const import ATTR_FORECAST, ATTR_FORECAST_TYPE
from homeassistant.core import ServiceCall, SupportsResponse
from homeassistant.helpers.typing import ServiceResponse

FORECAST_HOURLY = [
    {
        "datetime": "2026-09-16T15:00:00+00:00",
        "temperature": 26.0,
        "humidity": 55,
    },
    {
        "datetime": "2026-09-16T16:00:00+00:00",
        "temperature": 24.0,
        "humidity": 60,
    },
    {
        "datetime": "2026-09-16T17:00:00+00:00",
        "temperature": 23.0,
        # humidity missing — must be skipped
    },
]


def async_mock_forecasts(hass, forecast=FORECAST_HOURLY):
    """Register weather.get_forecasts for tests."""

    async def _handler(call: ServiceCall) -> ServiceResponse:
        return {WEATHER_ENTITY_ID: {"forecast": list(forecast)}}

    hass.services.async_register(
        "weather",
        "get_forecasts",
        _handler,
        supports_response=SupportsResponse.ONLY,
    )


@pytest.mark.parametrize(*WEATHER_YAML)
async def test_weather_forecast_attribute(hass, start_ha_weather):
    """Each sensor exposes projected values for complete forecast periods."""
    async_mock_forecasts(hass)
    async_set_weather(hass)  # trigger refresh
    await hass.async_block_till_done()

    heat = get_sensor(hass, SensorType.HEAT_INDEX)
    forecast = heat.attributes[ATTR_FORECAST]
    assert len(forecast) == 2
    assert forecast[0]["datetime"] == "2026-09-16T15:00:00+00:00"
    assert forecast[0]["temperature"] == 26.0
    assert forecast[0]["humidity"] == 55
    assert forecast[0]["value"] == pytest.approx(calculate_heat_index(26.0, 55.0))
    assert heat.attributes[ATTR_FORECAST_TYPE] == "hourly"


@pytest.mark.parametrize(*WEATHER_YAML)
async def test_sensor_mode_has_no_forecast_attribute(hass, start_ha):
    """Temperature/humidity devices do not expose forecast."""
    assert ATTR_FORECAST not in get_sensor(hass, SensorType.HEAT_INDEX).attributes


def test_forecast_is_unrecorded():
    """Forecast lists must not be written to the recorder."""
    from custom_components.thermal_comfort.sensor import SensorThermalComfort

    assert ATTR_FORECAST in SensorThermalComfort._unrecorded_attributes
```

`test_sensor_mode_has_no_forecast_attribute` uses the existing `start_ha` fixture from `tests/test_sensor.py` (`DEFAULT_TEST_SENSORS`). Import `DEFAULT_TEST_SENSORS` and parametrize:

```python
from .test_sensor import DEFAULT_TEST_SENSORS, LEN_DEFAULT_SENSORS, get_sensor

@pytest.mark.parametrize(*DEFAULT_TEST_SENSORS)
async def test_sensor_mode_has_no_forecast_attribute(hass, start_ha):
    assert ATTR_FORECAST not in get_sensor(hass, SensorType.HEAT_INDEX).attributes
```

If `start_ha_weather` runs before the mock service exists, forecasts will be empty on first setup. Either register the mock service in `start_ha_weather` before `async_setup_component`, or after setup call `async_set_weather` again once the service exists (as above). Prefer registering the mock **inside** `start_ha_weather` before setup so the initial fetch works:

```python
@pytest.fixture
async def start_ha_weather(hass, domains, config):
    async_set_weather(hass)
    async_mock_forecasts(hass)
    ...
```

Then `test_weather_current_values` still passes (forecast presence is extra). Split: `start_ha_weather` always registers an empty-forecast service so missing-service does not warn loudly; tests that need data call `async_mock_forecasts` with `FORECAST_HOURLY` and refresh.

Because `hass.services.async_register` cannot replace easily, register once in the fixture with a mutable list:

```python
@pytest.fixture
async def start_ha_weather(hass, domains, config):
    forecast_payload = []

    async def _handler(call: ServiceCall) -> ServiceResponse:
        return {WEATHER_ENTITY_ID: {"forecast": list(forecast_payload)}}

    hass.services.async_register(
        "weather", "get_forecasts", _handler, supports_response=SupportsResponse.ONLY
    )
    hass.data["test_weather_forecast_payload"] = forecast_payload
    async_set_weather(hass)
    ...
```

Tests that need forecasts:

```python
    hass.data["test_weather_forecast_payload"][:] = FORECAST_HOURLY
    async_set_weather(hass)
    await hass.async_block_till_done()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pytest tests/test_weather.py::test_weather_forecast_attribute tests/test_weather.py::test_forecast_is_unrecorded -v`

Expected: FAIL (`forecast` attribute missing / `_unrecorded_attributes` missing).

- [ ] **Step 3: Resolve forecast type and fetch forecasts**

Add to `DeviceThermalComfort`:

```python
    def _resolve_forecast_type(self, supported_features: int) -> str | None:
        """Return the forecast type to request, or None if none is usable."""
        requested = self._forecast_type
        flags = {
            FORECAST_TYPE_HOURLY: WEATHER_FORECAST_HOURLY,
            FORECAST_TYPE_DAILY: WEATHER_FORECAST_DAILY,
            FORECAST_TYPE_TWICE_DAILY: WEATHER_FORECAST_TWICE_DAILY,
        }
        if requested != FORECAST_TYPE_AUTO:
            if supported_features & flags[requested]:
                return requested
            _LOGGER.warning(
                "Weather entity %s does not support %s forecasts",
                self._weather_entity,
                requested,
            )
            return None
        for forecast_type in (
            FORECAST_TYPE_HOURLY,
            FORECAST_TYPE_TWICE_DAILY,
            FORECAST_TYPE_DAILY,
        ):
            if supported_features & flags[forecast_type]:
                return forecast_type
        return None

    async def async_refresh_forecasts(self) -> None:
        """Fetch weather forecasts and compute projected indices."""
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
            self._resolved_forecast_type = None
            self.extra_state_attributes.pop(ATTR_FORECAST_TYPE, None)
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
        except Exception:  # noqa: BLE001 — keep current sensors healthy
            _LOGGER.warning(
                "Could not fetch %s forecasts from %s",
                forecast_type,
                self._weather_entity,
                exc_info=True,
            )
            return

        periods = []
        if isinstance(response, dict):
            periods = list(
                (response.get(self._weather_entity) or {}).get("forecast") or []
            )

        unit = state.attributes.get(
            "temperature_unit", self.hass.config.units.temperature_unit
        )
        computed: dict[SensorType, list[dict]] = {
            sensor_type: [] for sensor_type in SENSOR_TYPES
        }
        for period in periods:
            datetime_iso = period.get("datetime")
            raw_temperature = period.get("temperature")
            raw_humidity = period.get("humidity")
            if datetime_iso is None:
                continue
            try:
                temp = util.convert(raw_temperature, float)
                humidity = float(raw_humidity)
                temperature = TemperatureConverter.convert(
                    temp, unit, UnitOfTemperature.CELSIUS
                )
            except (TypeError, ValueError):
                continue
            if not -89.2 <= temperature <= 56.7:
                continue
            if not 0 < humidity <= 100:
                continue
            for sensor_type in SENSOR_TYPES:
                result = calculate_sensor(sensor_type, temperature, humidity)
                extras = {}
                value = result
                if isinstance(result, tuple) and len(result) == 2:
                    value, extras = result
                    extras = dict(extras)
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
        self._resolved_forecast_type = forecast_type
        self.extra_state_attributes[ATTR_FORECAST_TYPE] = forecast_type
        await self.async_update_sensors(True)
```

Call `await self.async_refresh_forecasts()` at the end of `_new_weather_state` after updating current T/RH (including the unavailable path: do not clear forecasts on a brief unavailable; only skip the fetch). Call it from `_async_poll_update` when `self._weather_entity` is set:

```python
    async def _async_poll_update(self, now=None) -> None:
        """Periodic update used when polling is enabled."""
        if self._weather_entity:
            await self.async_refresh_forecasts()
        await self.async_update_sensors(True)
```

- [ ] **Step 4: Attach `forecast` on weather-mode sensors**

On `SensorThermalComfort`:

```python
class SensorThermalComfort(SensorEntity):
    """Representation of a Thermal Comfort Sensor."""

    _unrecorded_attributes = frozenset({ATTR_FORECAST})
```

In `async_update`, after setting `_attr_native_value`:

```python
        if self._device.weather_entity:
            self._attr_extra_state_attributes[ATTR_FORECAST] = (
                self._device.forecasts.get(self._sensor_type, [])
            )
        else:
            self._attr_extra_state_attributes.pop(ATTR_FORECAST, None)
```

Add properties on `DeviceThermalComfort`:

```python
    @property
    def weather_entity(self) -> str | None:
        """Return the configured weather entity, if any."""
        return self._weather_entity

    @property
    def forecasts(self) -> dict[SensorType, list[dict]]:
        """Return computed forecast values keyed by sensor type."""
        return self._forecasts
```

Initialize `self._forecasts = {}` in `__init__`. `extra_state_attributes` merge already includes device-level `forecast_type`.

Enum values in forecast `value` fields: Home Assistant serializes `StrEnum` as their string value. Tests may compare with the enum or with `str(value)`.

- [ ] **Step 5: Run weather, sensor, and config-flow tests**

Run:

```bash
pytest tests/test_weather.py tests/test_sensor.py tests/test_config_flow.py tests/test_calculations.py tests/test_init.py -v
```

Expected: PASS. Sensor-mode tests must not see a `forecast` attribute. Absolute humidity at 25 °C / 50 % RH is still `"11.5128065738593"`.

- [ ] **Step 6: Commit**

```bash
git add custom_components/thermal_comfort/sensor.py tests/test_weather.py
git commit -m "$(cat <<'EOF'
feat: project thermal comfort values from weather forecasts

Compute each index for forecast periods and expose them on a
non-recorded forecast attribute of the existing sensors.
EOF
)"
```

---

### Task 5: Documentation

**Files:**
- Modify: `documentation/yaml.md`
- Modify: `documentation/config_flow.md`

**Interfaces:**
- Consumes: YAML keys and config-flow steps from Tasks 2–3; forecast attribute shape from Task 4
- Produces: user-facing docs that match the implemented behavior

- [ ] **Step 1: Update YAML docs**

In `documentation/yaml.md`, add a weather example next to the existing sensor example:

```yaml
thermal_comfort:
  - sensor:
    - name: Outside
      weather_entity: weather.forecast_home
      forecast_type: auto
      unique_id: 7c2e0b5a-4d11-4f0c-9c3e-weather-outside
    - name: Living Room
      temperature_sensor: sensor.temperature_livingroom
      humidity_sensor: sensor.humidity_livingroom
      unique_id: 2f842c63-051a-4c49-9da2-4f04ee677514
```

Document:

- `weather_entity` (`string`, exclusive with the sensor pair): weather entity used for current temperature/humidity attributes and forecasts.
- `forecast_type` (`string`, optional, default `auto`): `auto`, `hourly`, `daily`, or `twice_daily`. Ignored when `weather_entity` is not set. `auto` prefers hourly, then twice-daily, then daily.
- `temperature_sensor` / `humidity_sensor` remain required **unless** `weather_entity` is set.
- Mixing `weather_entity` with either sensor on the same device is invalid.

Add a short "Forecast attribute" note: weather-mode sensors include `forecast`, a list of `{datetime, temperature, humidity, value}` (plus the same extra keys the current sensor already has). `forecast` is not recorded in history.

- [ ] **Step 2: Update config-flow docs**

In `documentation/config_flow.md`, replace the sentence that only mentions temperature and humidity:

- After naming the virtual device, choose **Temperature and humidity sensors** or **Weather entity**.
- Sensor mode: existing picker behavior and advanced-mode filtering.
- Weather mode: pick a `weather.*` entity and a forecast type (default Auto).
- Options: sensor devices still edit the two source sensors; weather devices edit the weather entity and forecast type. Source mode cannot be switched later — create a new device instead.

- [ ] **Step 3: Commit**

```bash
git add documentation/yaml.md documentation/config_flow.md
git commit -m "$(cat <<'EOF'
docs: describe weather entity input and forecast attributes

Document YAML and UI setup for weather-based thermal comfort,
including exclusive sources and the forecast extra attribute.
EOF
)"
```

---

## Plan self-review

1. **Spec coverage:** Weather current values (Task 2), YAML exclusive source (Task 2), config flow (Task 3), forecasts + unrecorded attribute (Task 4), docs (Task 5), calculation reuse (Task 1). Non-goals (no new indices, no mode switch, no recorder writes) are respected.
2. **Placeholders:** None. Formulas, schemas, unique IDs, service call, and test names are specified.
3. **Types:** `CONF_WEATHER_ENTITY`, `CONF_FORECAST_TYPE`, `ATTR_FORECAST`, `calculate_sensor(sensor_type, temperature, humidity)`, `DeviceThermalComfort.forecasts`, unique ID `weather-{id}` are named the same in every task.
