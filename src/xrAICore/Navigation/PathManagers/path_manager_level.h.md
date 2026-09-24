# src/xrAICore/Navigation/PathManagers/path_manager_level.h

> Declares the search policy for a level's navigation mesh, implemented in [`path_manager_level_inline.h`](path_manager_level_inline.h.md).

**Needs** — [`path_manager_generic.h`](path_manager_generic.h.md) · [`path_manager_params.h`](path_manager_params.h.md) · [`../level_graph.h`](../level_graph.h.md) · [`path_manager_level_inline.h`](path_manager_level_inline.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_level_flooder.h`](path_manager_level_flooder.h.md) · [`path_manager_level_inline.h`](path_manager_level_inline.h.md) · [`path_manager_level_nearest_vertex.h`](path_manager_level_nearest_vertex.h.md) · [`path_manager_level_straight_line.h`](path_manager_level_straight_line.h.md)
**Tier floor** — T2: integer cell arithmetic and a comparison per vertex.

## Purpose

Declares the surface implemented in
[`path_manager_level_inline.h`](path_manager_level_inline.h.md): the policy used for every
ordinary route across the fine navigation mesh of the loaded level. It is the most-executed
search configuration in the engine.

## State

The declared members are the working set of the cost model: the cell coordinates of the start,
of the goal and of the vertex currently being expanded; the mesh's cell size and its square;
and a cached pointer to the vertex currently being expanded. All are derived in `setup` and
`init` and refreshed per expansion — see the implementation.

## Exported units

- `setup(...)` — as the base, plus caching the mesh's cell size
- `init()` — unpack the start and goal cell coordinates once, before the first expansion
- `evaluate(from, to, edge)` — one cell's width, for every edge
- `estimate(vertex)` — a weighted Manhattan distance in cells
- `is_goal_reached(vertex)` — goal identity, and the hook that refreshes the expansion state
- `is_limit_reached(iterations)` — the base budgets
- `is_accessible(vertex)` — the mesh's validity-and-access mask
- `begin` / `get_value` — the neighbour walk, driven from the cached vertex
