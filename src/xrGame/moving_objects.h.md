# src/xrGame/moving_objects.h

> Declares the world's obstacle-avoidance system — the index of moving creatures and the solver that decides which of two converging creatures waits.

**Needs** — [`quadtree.h`](quadtree.h.md) · [`obstacles_query.h`](obstacles_query.h.md) · [`moving_objects_inline.h`](moving_objects_inline.h.md)
**Used by** — [`ai_obstacle.cpp`](ai_obstacle.cpp.md) · [`ai_obstacle.h`](ai_obstacle.h.md) · [`ai_space.cpp`](ai_space.cpp.md) · [`dynamic_obstacles_avoider.cpp`](dynamic_obstacles_avoider.cpp.md) · [`moving_object.cpp`](moving_object.cpp.md) · [`moving_objects.cpp`](moving_objects.cpp.md) · [`moving_objects_dynamic.cpp`](moving_objects_dynamic.cpp.md) · [`moving_objects_dynamic_collision.cpp`](moving_objects_dynamic_collision.cpp.md) · [`moving_objects_impl.h`](moving_objects_impl.h.md) · [`moving_objects_inline.h`](moving_objects_inline.h.md) · [`moving_objects_static.cpp`](moving_objects_static.cpp.md) · [`stalker_movement_manager_obstacles_path.cpp`](stalker_movement_manager_obstacles_path.cpp.md) · [`static_obstacles_avoider.cpp`](static_obstacles_avoider.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `moving_objects`, one per world. It owns a spatial index of every living creature
that can move, and the machinery that predicts their paths forward a second, finds where
those predictions overlap, and assigns each involved creature either *move* or *wait*.
Substance is in four implementation twins, split by what is being avoided:
[`moving_objects.cpp`](moving_objects.cpp.md) (index lifecycle),
[`moving_objects_static.cpp`](moving_objects_static.cpp.md) (level furniture),
[`moving_objects_dynamic.cpp`](moving_objects_dynamic.cpp.md) (the solver) and
[`moving_objects_dynamic_collision.cpp`](moving_objects_dynamic_collision.cpp.md) (the
pairwise decision).

Exported units:

- `moving_objects` — holds the spatial index and six scratch containers reused across
  frames.
- `possible_actions` — a two-bit vocabulary: *the first may wait for the second*, *the
  second may wait for the first*. A collision is always resolved into exactly one of these.
- The collision record chain — a collision is a pair of records; a collision-action is that
  pair plus the chosen action; a timed collision-action is that plus the time along the
  prediction at which the overlap occurs. The solver works on a list of the last.
- `on_level_load` — rebuild the index for the newly loaded level's bounds.
- `register_object` / `unregister_object` / `on_object_move` — index membership.
- `query_action_dynamic` — the solver: run a transitive sweep from one creature and assign
  actions to everyone it reaches.
- `query_action_static` — fill one creature's static obstacle set from the level furniture
  its predicted path would hit.
- `collisions` — the previous frame's resolved collisions, which the solver reads to keep
  its decisions stable.
- `clear` — forget the previous frame's collisions.
