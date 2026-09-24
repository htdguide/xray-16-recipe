# src/xrGame/ai/monsters/states/monster_state_hear_danger_sound.h

> Declares the reaction to a frightening sound, implemented in
> [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md)
**Used by** — [`bloodsucker_state_manager.cpp`](../bloodsucker/bloodsucker_state_manager.cpp.md) · [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md) · [`burer_state_manager.cpp`](../burer/burer_state_manager.cpp.md) · [`cat_state_manager.cpp`](../cat/cat_state_manager.cpp.md) · [`chimera_state_manager.cpp`](../chimera/chimera_state_manager.cpp.md) · [`controller_state_manager.cpp`](../controller/controller_state_manager.cpp.md) · [`flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) · [`fracture_state_manager.cpp`](../fracture/fracture_state_manager.cpp.md) · [`poltergeist_state_manager.cpp`](../poltergeist/poltergeist_state_manager.cpp.md) · [`pseudodog_state_manager.cpp`](../pseudodog/pseudodog_state_manager.cpp.md) · [`pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md) · [`snork_state_manager.cpp`](../snork/snork_state_manager.cpp.md) · [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md) · [`tushkano_state_manager.cpp`](../tushkano/tushkano_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the top-level behaviour a creature runs when the senses report a sound classified as
dangerous — gunfire, an explosion, a larger predator. Registered directly in most creature state
managers as one of the competing top-level behaviours.

## `CStateMonsterHearDangerousSound`

- **construct** — register the four leaves: bolt for cover, face the open ground, cower, and go home
- **reselect** — choose among them
- **setup** — fill each leaf's parameters

Stateless beyond the base's leaf bookkeeping. Contracts are in the implementation twin.
