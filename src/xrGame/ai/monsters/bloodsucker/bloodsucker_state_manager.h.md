# src/xrGame/ai/monsters/bloodsucker/bloodsucker_state_manager.h

> Declares the bloodsucker's brain: the shared creature brain plus a feed test and a seize hook.

**Needs** — [`monster_state_manager.h`](../monster_state_manager.h.md) · [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md)
**Used by** — [`bloodsucker.cpp`](bloodsucker.cpp.md) · [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md). It is the shared creature brain specialised to this creature, and it adds exactly two things to that contract.

## `BloodsuckerStateManager`

- **execute** — the per-update selector: pick one global state, run it, remember it.
- **update** — the scheduled tick; a plain delegation to the shared brain.
- **check_feeding** — whether the feed should take over the creature right now.
- **seize_corpse** — attach a captured entity to the creature by a named bone so it can be dragged.
- **remove_links** — reference cleanup, delegated.

**Notes** — the brain holds no fields of its own. Everything it decides with is read from the creature and from the registered states.
