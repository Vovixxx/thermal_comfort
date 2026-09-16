# Future request — Weather forecasts for comfort indices

**Date:** 2026-09-16  
**Status:** Shelf only — do not implement  
**Depends on:** Stage 1 (current weather source) shipped and used  
**Parent:** `docs/superpowers/specs/2026-09-16-thermal-comfort-roadmap.md`

## Intent

Met.no / Open-Meteo give current T/H and a weather forecast. They do not give this integration's derived sensors — especially the **text** ones ("a bit dry for some", "probable frost", "comfortable"). Once Stage 1 can read current weather attributes, the next upgrade is to run the **same** calculations on forecast periods so you can see what to expect, not only what it is now.

Numeric values matter less than those perception sensors. Exposure should stay convenient (dashboards, automations) without inventing a new card system in the first forecast PR.

## Why this is not Stage 1

Forecasts need a fetch path (`weather.get_forecasts`), a place to put lists of future values, and a viewing choice. That is a second product decision. Stage 1 should prove weather-as-source with the sensors people already know. Forecasts get their own spec once that feels right — more ideas will show up in use.

## Ideas already on this shelf (not decided)

- `forecast` list on each existing sensor (More Info / automations)
- `forecast_next` (+ datetime) as the glanceable "what to expect"
- Hourly vs daily: auto-pick hourly if the weather entity supports it, else daily; no picker unless we learn we need one
- Perception sensors as the headline; numerics come along for free if the loop is shared
- Extract formulas into `calculations.py` only when this stage needs to run them for many (T, H) pairs
- **Not this shelf:** expected vs actual (forecast vs what later happened). Park until forecasts exist. Current sensor history already records reality.

## Open questions (answer in a future design)

1. Is "next period" (often next hour) the right "what to expect", or do people want tomorrow's daily summary?
2. Do we attach forecasts to all 16 sensors or only perceptions?
3. How do we show a horizon without a custom card? (More Info, `forecast_next` on an entities card, optional Markdown example)
4. The stock Weather forecast card only accepts `weather.*`. We are **not** adding a Thermal Comfort weather entity just to feed that card unless a later spec argues for it.
5. Recorder: future-valued lists should not bloat history (`_unrecorded_attributes`).

## Suggested shape when this shelf is opened

- New spec + plan + PR, after Stage 1.
- Reuse Stage 1's weather listener; add `get_forecasts` there.
- Share calculation code; do not duplicate formulas.
- Keep indoor sensor-mode devices free of forecast attributes.

## Out of scope even then

- Device linking (Stage 2)
- Custom Lovelace cards
- Expected vs actual
- Changing formulas
