# src/xrGame/ai/monsters/zombie/zombie_state_manager.h

> Declares the zombie's behaviour tree root.

**Needs** — [`monster_state_manager.h`](../monster_state_manager.h.md) · [`zombie_state_manager.cpp`](zombie_state_manager.cpp.md)
**Used by** — [`zombie.cpp`](zombie.cpp.md) · [`zombie_state_manager.cpp`](zombie_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`zombie_state_manager.cpp`](zombie_state_manager.cpp.md): a constructor that registers the
zombie's six behaviours and one per-tick selection function.

## Exported units

- **construction** — registers the six behaviours.
- **the selection function** — the per-tick decision, including the suppression check that
  freezes the mind during a feigned death.
