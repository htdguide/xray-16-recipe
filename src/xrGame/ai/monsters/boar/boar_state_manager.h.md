# src/xrGame/ai/monsters/boar/boar_state_manager.h

> Declares the boar's top-level state selector.

**Needs** — [`monster_state_manager.h`](../monster_state_manager.h.md) · [`boar_state_manager.cpp`](boar_state_manager.cpp.md)
**Used by** — [`boar.cpp`](boar.cpp.md) · [`boar_state_manager.cpp`](boar_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`boar_state_manager.cpp`](boar_state_manager.cpp.md). A creature's state manager is the root of its behaviour tree; it adds no data of its own and overrides only the tick.

## `BoarStateManager`

Overrides `execute` — one pass of "decide which top-level state applies, then run it". Everything else is the shared creature state-manager contract.
