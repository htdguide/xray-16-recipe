# src/xrGame/static_obstacles_avoider_inline.h

> Binding, clearing, and the four set accessors.

**Needs** — [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md)
**Used by** — [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md)
**Tier floor** — T3: assignment and field access.

## Purpose

Small members kept out of the header. A rebuild puts them on the type and deletes this file.
Only two carry anything.

## `construct(movement_manager, failed_to_build_path)`

**Contract** — supplies the two bindings the avoider cannot get at construction, because the
movement manager holds this object by value and so must exist first. Two-phase construction
is the incidental part; the decision that survives is that the avoider *reads* the movement
layer's path-failure state continuously rather than being notified of it.

## `clear`

**Contract** — empties all four obstacle sets. All four, not just the active pair: leaving
the previous-cycle set populated would make the next cycle's change test compare against a
world that no longer exists and conclude, wrongly, that nothing had changed.

## The accessors

**Contract** — `need_path_to_rebuild` reports whether this cycle changed the set the
pathfinder should use; `active_query`, `inactive_query` and `current_iteration` hand out the
sets themselves. The active set is what the pathfinder is given; the other two are exposed
for the dynamic variant and for diagnostics.
