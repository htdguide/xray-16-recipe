# src/xrGame/ai/monsters/states/monster_state_rest.h

> Declares the peacetime behaviour — everything a creature does when nothing is happening —
> implemented in [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md) · [`../../../EntityCondition.h`](../../../EntityCondition.h.md)
**Used by** — [`bloodsucker_state_manager.cpp`](../bloodsucker/bloodsucker_state_manager.cpp.md) · [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md) · [`burer_state_manager.cpp`](../burer/burer_state_manager.cpp.md) · [`cat_state_manager.cpp`](../cat/cat_state_manager.cpp.md) · [`chimera_state_manager.cpp`](../chimera/chimera_state_manager.cpp.md) · [`controller_state_manager.cpp`](../controller/controller_state_manager.cpp.md) · [`flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) · [`fracture_state_manager.cpp`](../fracture/fracture_state_manager.cpp.md) · [`poltergeist_state_rest.h`](../poltergeist/poltergeist_state_rest.h.md) · [`pseudodog_state_manager.cpp`](../pseudodog/pseudodog_state_manager.cpp.md) · [`pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md) · [`snork_state_manager.cpp`](../snork/snork_state_manager.cpp.md) · [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md) · [`tushkano_state_manager.cpp`](../tushkano/tushkano_state_manager.cpp.md) · _and 1 more_
**Tier floor** — T3: a declaration

## Purpose

Names the top-level behaviour that runs when a creature has no enemy, no hit, no alarming sound and
nothing to eat. It is the default state of almost every creature in the world at almost every
moment, so it is the behaviour a player sees most and the one the alife simulation spends its time
in. Registered directly in nearly every creature state manager.

It carries one compiled-in number, the sixty-second period that governs the alternation between
idling and wandering, and two timestamps.

## `CStateMonsterRest`

- **construct** — register nine leaves
- **enter** / **leave** — turn the anomaly detector on and off
- **execute** — a hand-written priority cascade, replacing the base's select-and-run machinery

The declared field recording the time of the last play bout is **dead**: written once on entry and
never read. Contracts, and the two registered leaves that are never selected, are in the
implementation twin.
