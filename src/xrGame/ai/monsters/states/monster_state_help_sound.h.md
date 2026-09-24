# src/xrGame/ai/monsters/states/monster_state_help_sound.h

> Declares the answer-a-packmate's-call behaviour, implemented in
> [`monster_state_help_sound_inline.h`](monster_state_help_sound_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`monster_state_help_sound_inline.h`](monster_state_help_sound_inline.h.md)
**Used by** — [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md) · [`cat_state_manager.cpp`](../cat/cat_state_manager.cpp.md) · [`dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) · [`flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) · [`pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md) · [`snork_state_manager.cpp`](../snork/snork_state_manager.cpp.md) · [`monster_state_help_sound_inline.h`](monster_state_help_sound_inline.h.md) · [`tushkano_state_manager.cpp`](../tushkano/tushkano_state_manager.cpp.md) · [`zombie_state_manager.cpp`](../zombie/zombie_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the top-level behaviour a creature runs when the senses report a distress call from its own
kind: run to the exact spot it came from, then look around. Registered directly in most creature
state managers.

## `CStateMonsterHearHelpSound`

- **construct** — register two leaves: run to the call, and look around
- **is_startable** — a call was heard, and it came from inside the home region if there is one
- **reselect** — run, then look, then stop offering a leaf at all
- **is_finished** — when the reselection declines to choose
- **setup** — fill each leaf's parameters

Stateless. Contracts are in the implementation twin.
