# src/xrGame/moving_object.h

> Declares the per-creature record the obstacle-avoidance system tracks, implemented in [`moving_object.cpp`](moving_object.cpp.md).

**Needs** — [`entity_alive.h`](entity_alive.h.md) · [`obstacles_query.h`](obstacles_query.h.md) · [`moving_object_inline.h`](moving_object_inline.h.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`dynamic_obstacles_avoider.cpp`](dynamic_obstacles_avoider.cpp.md) · [`moving_object.cpp`](moving_object.cpp.md) · [`moving_object_inline.h`](moving_object_inline.h.md) · [`moving_objects.cpp`](moving_objects.cpp.md) · [`moving_objects_dynamic.cpp`](moving_objects_dynamic.cpp.md) · [`moving_objects_dynamic_collision.cpp`](moving_objects_dynamic_collision.cpp.md) · [`moving_objects_static.cpp`](moving_objects_static.cpp.md) · [`static_obstacles_avoider.cpp`](static_obstacles_avoider.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `moving_object`, the shadow record one living creature has inside the world's
obstacle-avoidance system. It carries the creature's last-indexed position (which lags the
creature's real one until the index is told to update), the decision the avoidance system
last made for it, and the two obstacle sets that decision produced. Substance is split
between [`moving_object.cpp`](moving_object.cpp.md) and
[`moving_object_inline.h`](moving_object_inline.h.md).

Exported units:

- `action_type` — the decision vocabulary: move, wait, four sidestep directions, follow.
  Only move and wait are ever produced; see the inline twin.
- `moving_object(creature)` — constructing one registers it with the world's index.
- destructor — unregisters it. The pairing is the invariant the whole system rests on.
- `on_object_move` — tell the index this creature has moved, so it can be reindexed.
- `update_position` — copy the creature's current position into the record.
- `position` / `radius` / `id` / `object` — the indexed position, the creature's radius, its
  name, and the creature itself.
- `predict_position` / `target_position` — forwarded to the creature's movement manager:
  where it will be in *t* seconds, and where it is ultimately going.
- `ignore` / `ignored` — one creature may be exempted from this one's obstacle tests, plus
  the creature itself always is.
- `action` (two setters and a getter) / `action_position` / `action_frame` / `action_time` —
  the current decision, where it was taken, on which frame and at what wall-clock time.
- `static_query` / `dynamic_query` — the two accumulated obstacle sets, one for level
  furniture and one for other creatures.
