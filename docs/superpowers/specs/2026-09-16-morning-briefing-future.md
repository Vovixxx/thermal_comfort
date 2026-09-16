# Future request — Morning briefing (clothes and rain)

**Date:** 2026-09-16  
**Status:** Shelf only — do not implement  
**Depends on:** Stage 1 (outdoor current), Stage 3 (forecasts) for “take a raincoat today”  
**Parent:** `docs/superpowers/specs/2026-09-16-thermal-comfort-roadmap.md`

## Intent

Home Assistant is an assistant for a home. In the morning it should be able to say, in one line, what the day asks of a person: take a raincoat, take an umbrella, dress warm — it is freezing — or it is fine. That is not a new dew-point formula. It is **folding already-computed comfort and weather into one result**.

This is the same pattern as the window advisor: many sources in, one suggestion out. Different question (what to wear / carry, not what to do with windows).

## Why this is not Stage 1

It needs outdoor comfort **and** a forecast (rain later, cold this afternoon). It is also copy and policy (“umbrella vs raincoat”) rather than an index. Stage 1 only makes outdoor *current* sensors exist.

A template sensor on top of Stage 1+3 data may be enough for a long time. This shelf is for a built-in briefing sensor if that stays painful.

## Inputs (likely)

- Outdoor Thermal Comfort perceptions (heat, humidity, frost) — current and forecast
- Weather entity: condition, precipitation probability, temperature (already on the weather device)
- Time of day (morning snapshot vs all-day rolling advice)
- Optional: user locale / unit system for the text

Indoor comfort is probably **not** the main input (you do not pick a coat from the living-room dew point).

## Output (not decided)

- One sensor, state = short advice enum or sentence
- Attributes: reasons, the forecast window used (today / next 8 hours), maybe “peak cold” / “rain at 15:00”
- A notification is an automation on that sensor, not a required integration feature

## Open questions (answer in a future design)

1. Enum (`take_umbrella`, `take_raincoat`, `dress_warm`, `dress_light`) vs a single localized sentence?
2. Morning-only (state updates at 06:00) vs continuous “if you leave now”?
3. Rain vs thermal comfort: who wins when it is both cold and wet?
4. Is this still Thermal Comfort, or a tiny “home briefing” helper that *reads* Thermal Comfort + weather? If the latter, it may not belong in this repo.
5. More ideas will show up (school run, evening, “windows + coat” combined briefing). Keep this shelf about **leave-the-house clothing/rain**. Combine later if the outputs want to merge.

## Suggested shape when this shelf is opened

- Own spec, plan, and PR, after forecasts are real.
- Decide repo boundary first: extra sensor type here vs a separate integration that only consumes these entities.
- No LLM requirement; rules on indices + precipitation are enough for v1 of this shelf.

## Out of scope even then

- Window open/close (separate shelf)
- Chatty daily newspaper / LLM summary
- Expected vs actual
