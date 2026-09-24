# src/xrGame/ai/monsters/tushkano/tushkano_state_manager.h

> Declares the tushkano's behaviour tree root.

**Needs** — [`monster_state_manager.h`](../monster_state_manager.h.md) · [`tushkano_state_manager.cpp`](tushkano_state_manager.cpp.md)
**Used by** — [`tushkano.cpp`](tushkano.cpp.md) · [`tushkano_state_manager.cpp`](tushkano_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`tushkano_state_manager.cpp`](tushkano_state_manager.cpp.md). The behaviour tree root for
a creature is always the same shape — a constructor that registers the states this creature
can be in, and one selection function that runs every tick — so this header carries nothing
but those two.

## Exported units

- **construction** — registers the tushkano's eight behaviours.
- **the selection function** — the per-tick decision; see the implementation twin.
