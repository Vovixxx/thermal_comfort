# Future request — Device linking (expand the source device)

**Date:** 2026-09-16  
**Status:** Shelf only — do not implement  
**Depends on:** Stage 1 (weather as T/H source) in use, plus a dedicated design pass  
**Parent:** `docs/superpowers/specs/2026-09-16-thermal-comfort-roadmap.md`

## Intent

Thermal Comfort is not a second gadget. It is extra measurements on something you already have:

- A **weather** device (Met.no / Open-Meteo) that already carries a weather entity
- A **home climate** device that already carries temperature and humidity entities

Today those comfort sensors sit on a separate virtual device named "Thermal Comfort". From the user's point of view that split is fake: T, H, dew point, and "comfortable" are one story. The later product should feel like a plugin that **expands the source device**. New sensors would live on that device, stay linked to it, and take its name by default (still overridable at setup).

This composes with **global vs zones**: the house outdoor may be weather-backed or sensor-backed; rooms stay satellites. Linking is about *where entities live in the registry*, not about which outdoor a room compares to.

## Why this is not Stage 1

Home Assistant devices belong to the integration that registered them. Putting custom-component entities onto another integration's device (or merging two source devices when T and H are not on the same physical device) needs device-registry research, unique-id policy, and a migration for every existing config entry. Doing it in the same PR as "read weather attributes" would turn a small additive feature into a rewrite.

## Open questions (answer in a future design, not now)

1. **Same device vs via-device.** HA can attach entities to an existing device's identifiers, or create a child device with `via_device`. Which one actually shows up as "on" the weather device in the UI?
2. **Two indoor sources.** Temperature and humidity are often one device, but not always. If they differ, which device gets the comfort sensors?
3. **Name.** Default to the source device's name; allow override at setup. What happens when the source device is renamed later?
4. **Migration.** Existing virtual devices must move entities without breaking automations (entity_id / unique_id).
5. **YAML.** How does YAML name and device assignment work if there is no config-entry device?
6. **Multiple expansions.** Two Thermal Comfort configs on the same weather entity (e.g. different enabled sensor subsets).

## Suggested shape when this shelf is opened

- One future spec, one implementation plan, one PR.
- Prototype on a throwaway install: can a custom component add sensors to `weather.forecast_home`'s device at all?
- Only then design migration. If HA cannot put foreign entities on that device, fall back to `via_device` (child device under the weather/climate device) — still better than an unrelated virtual box, still not a rewrite of the math.

## Out of scope even then

- Changing index formulas
- Forecasts (Stage 3)
- Custom cards
