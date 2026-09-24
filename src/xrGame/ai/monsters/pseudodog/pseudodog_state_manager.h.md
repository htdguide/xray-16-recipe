# src/xrGame/ai/monsters/pseudodog/pseudodog_state_manager.h

> Declares the pseudodog's brain: seven states and a plain priority selector.

**Needs** — [`pseudodog_state_manager.cpp`](pseudodog_state_manager.cpp.md) · [`../monster_state_manager.h`](../monster_state_manager.h.md)
**Used by** — [`pseudodog.cpp`](pseudodog.cpp.md) · [`pseudodog_state_manager.cpp`](pseudodog_state_manager.cpp.md) · [`psy_dog_state_manager.cpp`](psy_dog_state_manager.cpp.md) · [`psy_dog_state_manager.h`](psy_dog_state_manager.h.md)
**Tier floor** — T3: a priority selector over a registered state set

## Purpose

Declares the surface implemented in
[`pseudodog_state_manager.cpp`](pseudodog_state_manager.cpp.md). It is also the base of
[`psy_dog_state_manager.h`](psy_dog_state_manager.h.md), which is why `execute` is overridable
and why the state registrations happen in the constructor rather than in a sealed routine — a
derived brain adds its own state and then defers to this one.

## Exported units

- construction — registers the seven states.
- `execute` — the selector.
- `forget_entity` — forwards to the base.
