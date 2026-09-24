# src/xrGame/dynamic_obstacles_avoider.h

> Declares the moving-obstacle avoider implemented in [`dynamic_obstacles_avoider.cpp`](dynamic_obstacles_avoider.cpp.md).

**Needs** — [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md) · [`dynamic_obstacles_avoider_inline.h`](dynamic_obstacles_avoider_inline.h.md)
**Used by** — [`dynamic_obstacles_avoider.cpp`](dynamic_obstacles_avoider.cpp.md) · [`dynamic_obstacles_avoider_inline.h`](dynamic_obstacles_avoider_inline.h.md) · [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `dynamic_obstacles_avoider`, the specialization of
[`static_obstacles_avoider`](static_obstacles_avoider.h.md) that avoids *other moving
creatures* rather than fixed level geometry. Substance in
[`dynamic_obstacles_avoider.cpp`](dynamic_obstacles_avoider.cpp.md).

Exported units:

- `query` — replaces the base's obstacle gathering with a question to the global
  moving-objects registry.
- `process_query` — narrows the base's two obstacle sets to what the registry just reported,
  and skips the whole pass while the creature has been told to stand still.
- `movement_enabled` — whether the registry's traffic-control decision currently lets this
  creature move.

**Notes** — the class adds no state of its own; it reinterprets the base's two obstacle sets
against a different source.
