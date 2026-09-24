# src/xrGame/ai/monsters/states/monster_state_hitted.h

> Declares the shot-from-nowhere reaction, implemented in
> [`monster_state_hitted_inline.h`](monster_state_hitted_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_hitted_inline.h`](monster_state_hitted_inline.h.md)
**Used by** — [`bloodsucker_state_manager.cpp`](../bloodsucker/bloodsucker_state_manager.cpp.md) · [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md) · [`burer_state_manager.cpp`](../burer/burer_state_manager.cpp.md) · [`cat_state_manager.cpp`](../cat/cat_state_manager.cpp.md) · [`controller_state_manager.cpp`](../controller/controller_state_manager.cpp.md) · [`dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) · [`flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) · [`fracture_state_manager.cpp`](../fracture/fracture_state_manager.cpp.md) · [`poltergeist_state_manager.cpp`](../poltergeist/poltergeist_state_manager.cpp.md) · [`pseudodog_state_manager.cpp`](../pseudodog/pseudodog_state_manager.cpp.md) · [`pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md) · [`snork_state_manager.cpp`](../snork/snork_state_manager.cpp.md) · [`monster_state_hitted_inline.h`](monster_state_hitted_inline.h.md) · [`tushkano_state_manager.cpp`](../tushkano/tushkano_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the top-level behaviour a creature runs when it has been hit but has no enemy — the case
where damage arrives from an attacker the creature cannot see. Registered directly in most creature
state managers; the chimera's manager has the registration commented out, so that species has no
hit reaction at all.

## `CStateMonsterHitted`

- **construct** — register three leaves: break away, stalk back, and go home
- **reselect** — go home if possible, otherwise alternate breaking away and stalking back

Stateless. Contracts are in the implementation twin.
