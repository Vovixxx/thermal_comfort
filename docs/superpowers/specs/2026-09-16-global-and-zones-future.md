# Future request — Global outdoor vs room zones

**Date:** 2026-09-16  
**Status:** Shelf only — do not implement  
**Depends on:** Stage 1 (weather *or* separate sensors as a source)  
**Parent:** `docs/superpowers/specs/2026-09-16-thermal-comfort-roadmap.md`

## Intent

There is one **global** outdoor comfort context for the house, and many **satellite** indoor zones (rooms). Later advice (windows, doors) hangs off the room and compares that room to the global outdoor — not to a second copy of the weather.

```text
                    GLOBAL (house outdoor)
                    weather entity  —or—  separate outdoor T + H sensors
                              │
                              ▼
                    Thermal Comfort (outside)
                    dew point, frost, "comfortable", …
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
     Living room          Bedroom              Kitchen
     (zone / satellite)   (zone)               (zone)
     indoor T + H         indoor T + H         indoor T + H
          │
          ├── window / door later (tied to this room)
          └── suggestions later (this room vs GLOBAL)
```

Weather is central, not exclusive. The global outdoor device must still be configurable with **separate sensors** (a real outdoor T/H pair, no weather integration). Weather is the convenient default when Met.no / Open-Meteo is what you have.

Zones are rooms. Each zone is its own Thermal Comfort device from indoor T+H. They **refer to** the global outdoor when they need “inside vs outside.” Doors and windows belong to a zone, not to the weather device.

Clothing / rain advice is house-wide: it reads the **global**, not a bedroom.

## Why this is not Stage 1

Stage 1 only adds weather as another way to feed **one** device’s T+H. It does not introduce a global pointer, zone membership, or door entities. Today you already get the shape by creating one “Outside” device and several room devices; they just do not know about each other yet.

## Stage 1 still true

- A device can use a weather entity **or** separate T and H sensors.
- Indoor rooms stay on separate sensors.
- No `global:` / `zone_of:` keys yet. Users name devices themselves (`Outside`, `Living Room`).

## Open questions (answer in a future design)

1. How does a zone point at the global — config option `outdoor_device:` / `global_entry_id:`, or implicit “the one weather-sourced device”?
2. Exactly one global per house, or allowed extras (garden vs street)?
3. Door vs window: same advisor, or door is “leave the room / house” and window is ventilation?
4. HA areas/floors: bind a zone to an Area so doors in that area attach automatically?
5. Device linking (Stage 2) vs this graph: linking is “sensors live on the physical device”; this is “rooms know the outdoor.” They compose; they are not the same PR.

## Suggested shape when this shelf is opened

- Own spec after Stage 1 is in daily use (one outdoor + a few rooms).
- Add an optional “this is the house outdoor / global” flag or a zone→global reference. Do not invent doors in that first linking PR unless it stays tiny.
- Window/door advisors consume that reference; they do not each pick a weather entity.

## Out of scope even then

- Rewriting formulas
- Forecasts (Stage 3) except as an input the global already has
- Morning digest
