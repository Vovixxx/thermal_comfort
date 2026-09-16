# Thermal Comfort — staged backlog

**Date:** 2026-09-16

This integration is named **Thermal Comfort**. Today that mostly means extra indices on T+H. The longer idea is still comfort of the home: indoor air, outdoor air, windows, what the weather asks of a person. Those later slices use the indices; they are not a different product name.

Ideas around weather input, device ownership, forecasts, windows, and clothing/rain advice showed up at once. They are **not one change**. They sit on separate shelves.

**Data first, then comfort advice.** Stages 1–3 make the numbers and texts exist. Window, door, and clothing advice sit on top and come last. Advice is on-demand (look at the sensor whenever), not a morning briefing.

## Shape of the house (target, not Stage 1)

- **Global (outdoor):** one house-wide Thermal Comfort. Fed by a **weather entity or separate outdoor T+H sensors**. Weather is central and convenient; separate sensors stay valid.
- **Satellites (zones / rooms):** indoor Thermal Comfort from that room’s T+H. Each zone **refers to the global** when it needs inside vs outside.
- **Doors and windows:** later, tied to their **room**, not to the weather device. Suggestions (open / close / wait) are per zone vs global.
- **Clothing / rain:** house-wide, reads the global.

Today you can already create one Outside device and several room devices. They do not point at each other yet. Wiring that graph is a later shelf.

Device linking (comfort sensors *on* the physical weather or climate device) is a separate later idea. It composes with this graph; it is not required for Stage 1.

## Mental model (where this is heading)

| Today | Intended direction |
| --- | --- |
| Weather entity unused; outdoor T/H copied into helpers | Global outdoor from weather **or** real outdoor sensors |
| Each Thermal Comfort entry is an island | Rooms are satellites of one global outdoor |
| Virtual “Thermal Comfort” device, unrelated to the source | Later: expand the source device (Stage 2); still not a rewrite of the math |

## Stages

| Stage | Shelf | Status | What it is |
| --- | --- | --- | --- |
| **1** | Weather as T/H source | **This work** | Optional weather entity **or** separate T+H sensors (unchanged). Current values only. Same virtual device. |
| **2** | Device linking | Future request | Attach comfort sensors to the source device; default name from that device. |
| **3** | Forecasts | Future request | Same indices for weather forecast periods; expose after Stage 1 exists. |
| **4** | Global + zones | Future request | One house outdoor (global); rooms as satellites that refer to it. |
| **5** | Window / door advisor | Future request | Per-room suggestion vs global; doors/windows tied to the room. |
| **6** | Clothing / rain advice | Future request | House-wide, on-demand, reads the global — not a morning digest. |
| — | Expected vs actual | Parked idea | Forecast vs what happened. Needs Stage 3. |

## Stage 1 — do this, merge this

Spec: `docs/superpowers/specs/2026-09-16-weather-entity-design.md`  
Plan: `docs/superpowers/plans/2026-09-16-weather-entity.md`

One additive feature on the current architecture. Weather **or** separate sensors. No global/zone graph. No formula changes. No entity moves. No forecast. Indoor YAML/UI stays valid.

## Later shelves (do not implement now)

- Device linking: `docs/superpowers/specs/2026-09-16-device-linking-future.md`
- Forecasts: `docs/superpowers/specs/2026-09-16-weather-forecast-future.md`
- Global + zones: `docs/superpowers/specs/2026-09-16-global-and-zones-future.md`
- Window advisor: `docs/superpowers/specs/2026-09-16-window-advisor-future.md`
- Clothing / rain advice: `docs/superpowers/specs/2026-09-16-clothing-advice-future.md`

Open those only when Stage 1 is in use and the next slice is chosen. Each gets its own design pass before code.

## Explicit non-goals for any near-term PR

- Full rewrite of the integration
- Custom Lovelace cards
- A Thermal Comfort `weather` platform just to feed the stock forecast card
- Forecast-type pickers, extra `*_forecast` entities, expected-vs-actual
- Window/door suggestions, clothing/rain advice, zone→global wiring, or any timed “briefing”
