# src/xrAICore/Navigation/PathManagers/path_manager_game_level.h

> Declares the policy that searches the cross-level graph for *any* vertex on a named level, implemented in [`path_manager_game_level_inline.h`](path_manager_game_level_inline.h.md).

**Needs** — [`path_manager_game.h`](path_manager_game.h.md) · [`path_manager_params_game_level.h`](path_manager_params_game_level.h.md) · [`../game_graph.h`](../game_graph.h.md) · [`path_manager_game_level_inline.h`](path_manager_game_level_inline.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_game_level_inline.h`](path_manager_game_level_inline.h.md)
**Tier floor** — T2: a termination test over a vertex attribute.

## Purpose

Declares the surface implemented in
[`path_manager_game_level_inline.h`](path_manager_game_level_inline.h.md). It refines the
cross-level routing policy for the case where the destination is a level rather than a vertex.

## Exported units

- `setup(...)` — as the routing policy, plus binding the request and invalidating its answer slot
- `estimate(vertex)` — zero; the heuristic is abandoned
- `is_goal_reached(vertex)` — is the cheapest open vertex on the wanted level
- `create_path(vertex)` — write the route only when the caller asked for one
