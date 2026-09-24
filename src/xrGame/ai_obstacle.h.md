# src/xrGame/ai_obstacle.h

> Declares the per-object obstacle: the set of navigation vertices one dynamic object blocks.

**Needs** — [`ai_obstacle.cpp`](ai_obstacle.cpp.md) · [`moving_objects.h`](moving_objects.h.md) · [`magic_box3.h`](magic_box3.h.md) · [`ai_obstacle_inline.h`](ai_obstacle_inline.h.md)
**Used by** — [`GameObject.cpp`](GameObject.cpp.md) · [`ai_obstacle.cpp`](ai_obstacle.cpp.md) · [`ai_obstacle_inline.h`](ai_obstacle_inline.h.md) · [`moving_objects_dynamic.cpp`](moving_objects_dynamic.cpp.md) · [`moving_objects_dynamic_collision.cpp`](moving_objects_dynamic_collision.cpp.md) · [`obstacles_query.cpp`](obstacles_query.cpp.md) · [`stalker_movement_manager_obstacles_path.cpp`](stalker_movement_manager_obstacles_path.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`ai_obstacle.cpp`](ai_obstacle.cpp.md) and
[`ai_obstacle_inline.h`](ai_obstacle_inline.h.md).

Exported units:

- **construction** from the game object whose body is the obstacle;
- **`area`** — the blocked navigation vertices, computed on demand;
- **`danger_area`** — a second, wider area; declared and never filled (see the cpp twin);
- **`crc`** — a checksum of the blocked set, for cheap change detection;
- **`on_move`** — invalidate, called when the object moves;
- **`inside`** by navigation vertex — is this vertex blocked;
- **`distance_to`** — distance from a point to the object's origin;
- **`min_box`** — the obstacle's oriented bounding box.

Privately it holds the six planes of that box, the computed vertex list, and the
validity flag that makes the computation lazy.
