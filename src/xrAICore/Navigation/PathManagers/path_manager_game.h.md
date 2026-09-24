# src/xrAICore/Navigation/PathManagers/path_manager_game.h

> Declares the search policy for the coarse cross-level graph, implemented in [`path_manager_game_inline.h`](path_manager_game_inline.h.md).

**Needs** — [`path_manager_generic.h`](path_manager_generic.h.md) · [`../game_graph.h`](../game_graph.h.md) · [`path_manager_game_inline.h`](path_manager_game_inline.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_game_inline.h`](path_manager_game_inline.h.md) · [`path_manager_game_level.h`](path_manager_game_level.h.md) · [`path_manager_game_vertex.h`](path_manager_game_vertex.h.md)
**Tier floor** — T2: a cost model over stored distances and world positions.

## Purpose

Declares the surface implemented in
[`path_manager_game_inline.h`](path_manager_game_inline.h.md): the policy used whenever the
graph being searched is the cross-level graph and the request is a plain route.

## Exported units

- `setup(...)` — as the base, plus caching the goal vertex so the heuristic need not look it up
- `evaluate(from, to, edge)` — the distance stored on the edge
- `estimate(vertex)` — straight-line distance from the vertex to the goal
- `is_limit_reached(iterations)` — never
- `is_accessible(vertex)` — the graph's own per-vertex enable flag
