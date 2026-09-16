# Future request — Window open/close advisor

**Date:** 2026-09-16  
**Status:** Shelf only — do not implement  
**Depends on:** Stage 1 (outdoor current from weather), indoor Thermal Comfort device, Stage 3 (forecasts) for timed advice  
**Parent:** `docs/superpowers/specs/2026-09-16-thermal-comfort-roadmap.md`

## Intent

Windows are manual. Home Assistant can still **suggest**: open them, close them, or close them in two hours. Indoor comfort, outdoor comfort, humidity, frost risk, and (later) the outdoor forecast are the inputs. The output is one clear instruction, not another wall of indices.

Examples of the kind of result:

- Open windows (inside is stuffy, outside is nicer and dry)
- Close windows (rain coming, or outdoor air is worse)
- Close windows in ~2 hours (fine now, forecast turns humid / cold / rainy)

## Why this is not Stage 1

It is a second product: compare **two** Thermal Comfort devices (room vs outside), add rules, and speak in actions instead of dew point. Timed advice needs forecasts. Building it now would mix a weather-entity reader with a home-assistant personality.

Until the data layers exist, a user can already approximate this with automations. This shelf is for a first-class suggestion sensor when that stops being enough.

## Inputs (likely)

- Indoor device: temperature, humidity, dew point / perception, maybe absolute humidity
- Outdoor device (Stage 1 weather source): same indices
- Weather condition / precipitation (from the weather entity)
- Outdoor forecast (Stage 3) for “in two hours”
- Optional: which window/cover entities exist — **advice can be text-only at first**; actually calling `cover.close` is a later choice

## Output (not decided)

- One sensor whose state is a short action: `open`, `close`, `close_later`, `leave`
- Extra attributes: reason (human text), `suggested_at` / `valid_until`, maybe `close_in_minutes`
- Notifications are optional; the sensor is enough for a dashboard or an automation the user already trusts

## Open questions (answer in a future design)

1. One advisor per indoor room, or one house-wide suggestion?
2. Do we ever actuate covers, or only suggest? Default: **suggest only** (windows are manual).
3. What beats what? Rain vs stuffy room vs outdoor pollen is policy, not math.
4. “In two hours” is forecast-driven. Without Stage 3, only current open/close.
5. Frost / mold / dew-on-glass: indoor dew point vs window surface is a different problem; do not smuggle it in without a spec.

## Suggested shape when this shelf is opened

- Own spec, plan, and PR.
- Require an indoor Thermal Comfort device + an outdoor (weather) one.
- Ship a suggestion sensor before any service that closes covers.

## Out of scope even then

- Rewriting core indices
- Device linking (Stage 2)
- Morning clothing briefing (separate shelf)
