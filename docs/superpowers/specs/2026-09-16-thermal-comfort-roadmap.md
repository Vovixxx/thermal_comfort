# Thermal Comfort — staged backlog

**Date:** 2026-09-16

This integration expands temperature + humidity into extra comfort sensors. Several ideas around weather, device ownership, and forecasts showed up at once. They are related, but they are **not one change**. Shipping them together would be a rewrite. They sit on separate shelves and get designed and built one at a time.

## Mental model (where this is heading)

| Today | Intended direction |
| --- | --- |
| A weather integration is a **device** with a weather entity | Thermal Comfort should feel like a **plugin on that device** |
| A temp/humidity peripheral is a **device** with T and H entities | Same: expand *that* device, not a second "Thermal Comfort" box |
| Thermal Comfort is its own virtual device | Later: sensors live on the source device, inherit its name (overridable at setup) |

That direction is real. It is also a Home Assistant device-registry problem, a migration problem, and a breaking-change problem. It is **not** Stage 1.

## Stages

| Stage | Shelf | Status | What it is |
| --- | --- | --- | --- |
| **1** | Weather as T/H source | **This work** | Optional weather entity instead of two helper sensors. Current values only. Same virtual device as today. |
| **2** | Device linking | Future request | Attach comfort sensors to the source device; default name from that device. |
| **3** | Forecasts | Future request | Compute the same indices for weather forecast periods; decide how to expose them after Stage 1 exists. |
| — | Expected vs actual | Parked idea | Compare what was forecast with what happened. Needs Stage 3 first. Do not design yet. |

## Stage 1 — do this, merge this

Spec: `docs/superpowers/specs/2026-09-16-weather-entity-design.md`  
Plan: `docs/superpowers/plans/2026-09-16-weather-entity.md`

One additive feature on the current architecture. No formula changes. No entity moves. No forecast. Indoor YAML/UI stays valid.

## Later shelves (do not implement now)

- Device linking: `docs/superpowers/specs/2026-09-16-device-linking-future.md`
- Forecasts: `docs/superpowers/specs/2026-09-16-weather-forecast-future.md`

Open those only when Stage 1 is in use and the next slice is chosen. Each gets its own design pass before code.

## Explicit non-goals for any near-term PR

- Full rewrite of the integration
- Custom Lovelace cards
- A Thermal Comfort `weather` platform just to feed the stock forecast card
- Forecast-type pickers, extra `*_forecast` entities, expected-vs-actual
