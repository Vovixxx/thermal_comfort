# Future request — Outdoor clothing / rain comfort advice

**Date:** 2026-09-16  
**Status:** Shelf only — do not implement  
**Depends on:** Stage 1 (global outdoor current), Stage 3 (forecasts) when the advice should look ahead  
**Parent:** `docs/superpowers/specs/2026-09-16-thermal-comfort-roadmap.md`

## Intent

This integration is called **Thermal Comfort**. The indices (dew point, “a bit dry for some”, frost risk) are the raw comfort language. Folding those with rain/cold into one practical line — take an umbrella, wear a raincoat, it is freezing, dress light — is the same job, aimed at a person about to go outside.

It is **not** a morning briefing, a daily newspaper, or a timed digest. The suggestion should be valid whenever you look: now, this afternoon, before a walk. A user can ping it from an automation at 07:00 if they want a morning ping; that is their automation, not this feature’s identity.

Same pattern as the window advisor: many sources in, one comfort result out. Different question (what the outdoor air asks of a body / what to carry, not what to do with windows).

## Why this is not Stage 1

It needs outdoor comfort sensors first, and forecasts if “it will rain later” should count. Copy and policy (“umbrella vs raincoat”) are a later design. Stage 1 only makes outdoor *current* indices exist.

## Inputs (likely)

- House **global** outdoor Thermal Comfort (weather *or* separate outdoor sensors)
- That global’s perceptions (heat, humidity, frost) — current, and forecast when Stage 3 exists
- Precipitation / condition from the weather entity when the global is weather-backed
- Horizon: “now” vs “the next few hours” — not a morning briefing

Indoor **zones** are not the input (a coat is not chosen from the living-room dew point).

## Output (not decided)

- One sensor, state = short advice enum or sentence (`take_umbrella`, `dress_warm`, …)
- Attributes: reasons, which hours were considered
- Any announcement (TTS, phone) is an automation on that sensor

## Open questions (answer in a future design)

1. Enum vs a localized sentence?
2. Default horizon (now only vs next N hours)?
3. Rain vs thermal comfort when it is both cold and wet?
4. Keep this as a Thermal Comfort sensor type (fits the name) vs a tiny consumer integration?
5. More slices will show up. This shelf stays **outdoor clothing / rain**. Do not merge with windows here.

## Suggested shape when this shelf is opened

- Own spec, plan, and PR, after current outdoor sensors exist (forecasts if looking ahead).
- No LLM; rules on indices + precipitation are enough.
- Do not build a “morning” schedule into the integration.

## Out of scope even then

- Window open/close (separate shelf)
- Morning-only / digest / LLM summary
- Expected vs actual
