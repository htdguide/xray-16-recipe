# src/xrGame/ai/monsters/burer/burer_state_manager.h

> Declares the burer's top-level state selector.

**Needs** — [`monster_state_manager.h`](../monster_state_manager.h.md) · [`burer_state_manager.cpp`](burer_state_manager.cpp.md)
**Used by** — [`burer.cpp`](burer.cpp.md) · [`burer_state_manager.cpp`](burer_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`burer_state_manager.cpp`](burer_state_manager.cpp.md).

## `BurerStateManager`

Overrides `execute` and, unusually for a state manager, `setup_substates` — because one of its top-level states is the generic "perform an action" state, which must be handed its parameters at selection time. Carries no data.
