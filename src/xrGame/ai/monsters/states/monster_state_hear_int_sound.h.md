# src/xrGame/ai/monsters/states/monster_state_hear_int_sound.h

> Declares the reaction to a merely interesting sound, implemented in
> [`monster_state_hear_int_sound_inline.h`](monster_state_hear_int_sound_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_hear_int_sound_inline.h`](monster_state_hear_int_sound_inline.h.md)
**Used by** — [`bloodsucker_state_manager.cpp`](../bloodsucker/bloodsucker_state_manager.cpp.md) · [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md) · [`burer_state_manager.cpp`](../burer/burer_state_manager.cpp.md) · [`cat_state_manager.cpp`](../cat/cat_state_manager.cpp.md) · [`chimera_state_manager.cpp`](../chimera/chimera_state_manager.cpp.md) · [`controller_state_manager.cpp`](../controller/controller_state_manager.cpp.md) · [`dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) · [`flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) · [`poltergeist_state_manager.cpp`](../poltergeist/poltergeist_state_manager.cpp.md) · [`pseudodog_state_manager.cpp`](../pseudodog/pseudodog_state_manager.cpp.md) · [`pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md) · [`snork_state_manager.cpp`](../snork/snork_state_manager.cpp.md) · [`monster_state_hear_int_sound_inline.h`](monster_state_hear_int_sound_inline.h.md) · [`zombie_state_manager.cpp`](../zombie/zombie_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the top-level behaviour for sounds the senses classify as worth investigating but not worth
fearing: footsteps, a door, another animal moving. Registered in most creature state managers as
one of the competing top-level behaviours.

## `CStateMonsterHearInterestingSound`

- **construct** — register two leaves: walk to the source, and look around there
- **reselect** — walk first if the walk leaf will accept, otherwise look; afterwards always look
- **setup** — fill each leaf's parameters
- **target position** (private) — where to walk, clamped to the home region

Stateless. Contracts are in the implementation twin.
