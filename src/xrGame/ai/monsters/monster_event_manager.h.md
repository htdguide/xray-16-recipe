# src/xrGame/ai/monsters/monster_event_manager.h

> Declares the creature's internal event bus: subscribe, unsubscribe, publish.

**Needs** — [`monster_event_manager_defs.h`](monster_event_manager_defs.h.md) · [`monster_event_manager.cpp`](monster_event_manager.cpp.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`monster_event_manager.cpp`](monster_event_manager.cpp.md)
**Tier floor** — T3: a map from event type to a list of callables

## Purpose

Declares the surface implemented in
[`monster_event_manager.cpp`](monster_event_manager.cpp.md). One of these lives inside each
creature; its subsystems — animation, movement, sound — use it to notify each other without
holding direct references.

## Exported units

- `subscribe(event_type, handler)` — register a handler. The same handler may be registered
  more than once for one event type, and will then be called more than once.
- `unsubscribe(event_type, handler)` — *mark* every matching registration for removal; the
  removal itself happens at the end of the next publish of that event type.
- `publish(event_type, payload)` — call every live handler for the type, then sweep the marked
  ones.

Teardown drops every registration.
