# src/xrAICore/Navigation/PathManagers/path_manager_level_nearest_vertex.h

> Declares the search for the reachable mesh vertex closest to an arbitrary point, implemented in [`path_manager_level_nearest_vertex_inline.h`](path_manager_level_nearest_vertex_inline.h.md).

**Needs** — [`path_manager_level.h`](path_manager_level.h.md) · [`path_manager_params_nearest_vertex.h`](path_manager_params_nearest_vertex.h.md) · [`path_manager_level_nearest_vertex_inline.h`](path_manager_level_nearest_vertex_inline.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_level_nearest_vertex_inline.h`](path_manager_level_nearest_vertex_inline.h.md)
**Tier floor** — T2: a squared-distance comparison per settled vertex.

## Purpose

Declares the surface implemented in
[`path_manager_level_nearest_vertex_inline.h`](path_manager_level_nearest_vertex_inline.h.md).
It is the flood fill with the collection step replaced by a running minimum.

## Exported units

- `setup(...)` — as the mesh policy, plus the start's cell coordinates, the radius in cells, the
  target position and an empty running best
- `is_goal_reached(vertex)` — never true; updates the running best
- `evaluate` / `estimate` — one cell width, and zero
- `is_accessible(vertex)` — the mesh's test, then the radius
- `is_limit_reached(iterations)` — the iteration and visited budgets only
- `create_path(vertex)` — nothing
