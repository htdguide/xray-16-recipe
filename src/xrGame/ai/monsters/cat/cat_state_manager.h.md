# src/xrGame/ai/monsters/cat/cat_state_manager.h

> Declares the cat's top-level state selector.

**Needs** — [`monster_state_manager.h`](../monster_state_manager.h.md) · [`cat_state_manager.cpp`](cat_state_manager.cpp.md)
**Used by** — [`cat.cpp`](cat.cpp.md) · [`cat_state_manager.cpp`](cat_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`cat_state_manager.cpp`](cat_state_manager.cpp.md).

## `CatStateManager`

Overrides `execute`. Carries one field, `last_rotation_jump_time`, which nothing reads — see the implementation's Notes.
