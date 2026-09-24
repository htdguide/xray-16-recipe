# src/xrGame/static_obstacles_avoider.h

> Declares the obstacle arbiter and marks the points a subclass may take over.

**Needs** — [`static_obstacles_avoider.cpp`](static_obstacles_avoider.cpp.md) · [`obstacles_query.h`](obstacles_query.h.md) · [`refreshable_obstacles_query.h`](refreshable_obstacles_query.h.md) · [`static_obstacles_avoider_inline.h`](static_obstacles_avoider_inline.h.md)
**Used by** — [`dynamic_obstacles_avoider.cpp`](dynamic_obstacles_avoider.cpp.md) · [`dynamic_obstacles_avoider.h`](dynamic_obstacles_avoider.h.md) · [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md) · [`static_obstacles_avoider.cpp`](static_obstacles_avoider.cpp.md) · [`static_obstacles_avoider_inline.h`](static_obstacles_avoider_inline.h.md)
**Tier floor** — T2: four sets and their access points.

## Purpose

Declares the surface implemented in
[`static_obstacles_avoider.cpp`](static_obstacles_avoider.cpp.md). Two things here are
decisions rather than declarations.

**Two of the operations are extension points**: the no-argument `query` and
`process_query`. The "static" in the name is the distinction — this one considers obstacles
by where they *are*, and a variant that considers where they are *going* replaces exactly
those two steps. So the split of the cycle into gather-then-decide is not organisational; it
is the seam a dynamic avoider is built on.

**The failure flag is borrowed, not owned.** The avoider holds a reference to the movement
layer's "I could not build a path" flag rather than being told about it. A rebuild may pass
it per call; what matters is that the freshness test consults it, because that is what
unsticks a stuck creature.

## Exported units

- `construct(movement manager, failed-to-build-path flag)` — the two bindings, supplied
  after construction because the movement manager owns this object by value.
- `on_before_query()` — start of cycle: retire and clear.
- `query(start, destination)` — gather the obstacles along an intended route.
- `query()` — gather the obstacles near the creature. An extension point.
- `process_query(may change path state)` — merge, test and adopt. An extension point.
- `update()` — the three steps in order.
- `remove_links(object)` — drop a destroyed object from all four sets.
- `clear()` — empty all four sets.
- `need_path_to_rebuild()`, `active_query()`, `inactive_query()`, `current_iteration()` —
  what the movement manager reads.
