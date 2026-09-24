# src/xrGame/ai/monsters/snork/snork_state_manager.h

> Declares the snork's brain: eight registered states and a selector that arms the pre-fight snarl.

**Needs** — [`snork_state_manager.cpp`](snork_state_manager.cpp.md) · [`../monster_state_manager.h`](../monster_state_manager.h.md)
**Used by** — [`snork.cpp`](snork.cpp.md) · [`snork_state_manager.cpp`](snork_state_manager.cpp.md)
**Tier floor** — T3: a priority selector over a registered state set

## Purpose

Declares the surface implemented in
[`snork_state_manager.cpp`](snork_state_manager.cpp.md). Nothing derives from this brain; the
virtual declarations follow the base's shape rather than serving an extension point.

## Exported units

- construction — registers the states.
- destruction — nothing; the container destroys the registered states.
- `execute` — the selector, plus the snarl arming.
- `remove_links` — forwards to the base cascade.
