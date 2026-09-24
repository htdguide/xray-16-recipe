# src/xrGame/ai/monsters/states/monster_state_panic.h

> Declares the flee-from-a-known-enemy behaviour, implemented in
> [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md)
**Used by** — [`bloodsucker_state_manager.cpp`](../bloodsucker/bloodsucker_state_manager.cpp.md) · [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md) · [`burer_state_manager.cpp`](../burer/burer_state_manager.cpp.md) · [`cat_state_manager.cpp`](../cat/cat_state_manager.cpp.md) · [`chimera_state_manager.cpp`](../chimera/chimera_state_manager.cpp.md) · [`controller_state_manager.cpp`](../controller/controller_state_manager.cpp.md) · [`flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) · [`fracture_state_manager.cpp`](../fracture/fracture_state_manager.cpp.md) · [`poltergeist_state_manager.cpp`](../poltergeist/poltergeist_state_manager.cpp.md) · [`pseudodog_state_manager.cpp`](../pseudodog/pseudodog_state_manager.cpp.md) · [`pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md) · [`snork_state_manager.cpp`](../snork/snork_state_manager.cpp.md) · [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md) · [`tushkano_state_manager.cpp`](../tushkano/tushkano_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the top-level behaviour a creature runs when it has an enemy and has decided not to fight it.
Registered directly in most creature state managers, alongside the attack behaviour it competes
with; which of the two the brain selects is a weighting decision made elsewhere.

## `CStateMonsterPanic`

- **construct** — register three leaves: bolt, face the open ground, and fall back into the territory
- **reselect** — territory first, otherwise alternate bolting and facing
- **check_force_state** — abandon the facing leaf the moment the enemy is visible or a hit lands
- **setup** — fill the facing leaf's parameters

Stateless. Contracts are in the implementation twin.
