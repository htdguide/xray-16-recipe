# src/xrAICore/Navigation/PathManagers/path_manager_game_vertex.h

> Declares the policy that confines a cross-level search to terrain a particular creature will cross, implemented in [`path_manager_game_vertex_inline.h`](path_manager_game_vertex_inline.h.md).

**Needs** — [`path_manager_game.h`](path_manager_game.h.md) · [`path_manager_params_game_vertex.h`](path_manager_params_game_vertex.h.md) · [`../game_graph.h`](../game_graph.h.md) · [`path_manager_game_vertex_inline.h`](path_manager_game_vertex_inline.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_game_vertex_inline.h`](path_manager_game_vertex_inline.h.md)
**Tier floor** — T2: a mask test per candidate vertex.

## Purpose

Declares the surface implemented in
[`path_manager_game_vertex_inline.h`](path_manager_game_vertex_inline.h.md). It refines the
cross-level routing policy with a per-creature terrain filter, and with the escape hatch that
keeps a creature standing on terrain it dislikes from being stranded.

## Exported units

- `setup(...)` — as the routing policy, plus binding the preference table and recording whether
  the start vertex itself passes the filter
- `is_accessible(vertex)` — the graph's enable flag, then the terrain filter, subject to the
  escape hatch
