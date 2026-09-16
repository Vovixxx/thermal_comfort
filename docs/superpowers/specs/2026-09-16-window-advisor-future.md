# Future request — Window open/close advisor

**Date:** 2026-09-16  
**Status:** Shelf only — do not implement  
**Depends on:** Stage 1 (outdoor current), indoor zone devices, Stage 4 (zone → global) if we do not hard-code the pair, Stage 3 (forecasts) for timed advice  
**Parent:** `docs/superpowers/specs/2026-09-16-thermal-comfort-roadmap.md`

## Intent

Windows (and later **doors**) are manual and belong to a **room**. Home Assistant can still **suggest**: open them, close them, or close them in two hours. The comparison is **this zone vs the house global outdoor**, not each window picking its own weather entity.

Indoor comfort, global outdoor comfort, humidity, frost risk, and (later) the outdoor forecast are the inputs. The output is one clear instruction, not another wall of indices.

Examples of the kind of result:

- Open windows (inside is stuffy, outside is nicer and dry)
- Close windows (rain coming, or outdoor air is worse)
- Close windows in ~2 hours (fine now, forecast turns humid / cold / rainy)

## Why this is not Stage 1

It is a second product: compare **two** Thermal Comfort devices (room vs outside), add rules, and speak in actions instead of dew point. Timed advice needs forecasts. Building it now would mix a weather-entity reader with a home-assistant personality.

Until the data layers exist, a user can already approximate this with automations. This shelf is for a first-class suggestion sensor when that stops being enough.

## Inputs (likely)

- Indoor **zone** device (the room)
- House **global** outdoor device (weather *or* separate outdoor sensors)
- Weather condition / precipitation when the global is weather-backed
- Outdoor forecast (Stage 3) for “in two hours”
- Window/door entities tied to that room later — **advice can be text-only at first**

## Output (not decided)

- One sensor whose state is a short action: `open`, `close`, `close_later`, `leave`
- Extra attributes: reason (human text), `suggested_at` / `valid_until`, maybe `close_in_minutes`
- Notifications are optional; the sensor is enough for a dashboard or an automation the user already trusts

## Open questions (answer in a future design)

1. Advisor is **per room** (zone vs global). House-wide rollup is a later extra, not the default.
2. Do we ever actuate covers, or only suggest? Default: **suggest only**.
3. Door vs window: same suggestion sensor, or door means “leave this room / house”?
4. What beats what? Rain vs stuffy room vs outdoor pollen is policy, not math.
5. “In two hours” is forecast-driven. Without Stage 3, only current open/close.
6. Frost / mold / dew-on-glass: indoor dew point vs window surface is a different problem; do not smuggle it in without a spec.

## Suggested shape when this shelf is opened

- Own spec, plan, and PR, after zones can refer to the global (Stage 4) or with an explicit outdoor device picker if Stage 4 is not done yet.
- One suggestion per indoor zone. Tie window/door entities to that zone when we add them.
- Ship a suggestion sensor before any service that closes covers.

## Out of scope even then

- Rewriting core indices
- Device linking (Stage 2)
- Clothing / rain advice (separate shelf)
