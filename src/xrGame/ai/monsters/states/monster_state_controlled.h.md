# src/xrGame/ai/monsters/states/monster_state_controlled.h

> Declares the puppet state: what a creature does while a controller is driving it.

**Needs** — [`monster_state_controlled_inline.h`](monster_state_controlled_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md) · [`dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) · [`flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) · [`pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md) · [`monster_state_controlled_inline.h`](monster_state_controlled_inline.h.md) · [`tushkano_state_manager.cpp`](../tushkano/tushkano_state_manager.cpp.md) · [`zombie_state_manager.cpp`](../zombie/zombie_state_manager.cpp.md)
**Tier floor** — T3: a two-way container keyed on an externally written task

## Purpose

Declares the surface implemented in
[`monster_state_controlled_inline.h`](monster_state_controlled_inline.h.md). Registered by every
creature a controller can take over, selected by that creature's brain whenever the
"under control" predicate is true, and holding no state of its own — the task it dispatches on
is written onto the creature by the controller, not by the brain.

## Exported units

- construction — registers the follow and attack children.
- `execute` — the dispatch.
- `remove_links` — forwards to the base cascade.
