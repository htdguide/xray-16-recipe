# src/xrAICore/Navigation/PathManagers/path_manager_game_level_inline.h

> "get me to that level" — a uniform-cost search over the cross-level graph that stops at the cheapest vertex belonging to a named level and reports which one it was.

**Needs** — [`path_manager_game_level.h`](path_manager_game_level.h.md) · [`path_manager_params_game_level.h`](path_manager_params_game_level.h.md) · [`path_manager_game_inline.h`](path_manager_game_inline.h.md)
**Used by** — [`path_manager_game_level.h`](path_manager_game_level.h.md)
**Tier floor** — T2: a comparison per expanded vertex.

## Purpose

The off-screen simulation moves entities between levels without knowing where on the
destination level they should arrive. This policy answers that: search outward from where the
entity is and take the first place the destination level is entered.

## `setup`

**Contract** — as the cross-level routing policy, and additionally keeps a reference to the
caller's request — which is an output record here, not just limits — and sets its answer slot
to the invalid marker so that a failed search cannot be mistaken for a successful one.

**Invariants** — the request must outlive the search; the policy writes into it.

## `estimate`

**Contract** — zero.

**Invariants** — this is what makes the answer meaningful. With a zero heuristic the driver
expands vertices in order of true cost from the start, so *the first vertex on the wanted level
to be expanded is the cheapest one to reach*. Restoring the parent policy's straight-line
heuristic would still find a vertex on that level, but no longer the nearest one — and there is
no goal position to aim a heuristic at anyway, since the goal is a set.

## `is_goal_reached`

**Contract** — reads the level attribute of the cheapest currently-open vertex and, when it
matches the wanted level, records that vertex in the request and stops the search.

**Notes** — it inspects the engine's current best vertex rather than the vertex it is handed.
The two are the same at every call site; reading through the workspace is redundant and a
rebuild should use the argument.

## `create_path`

**Contract** — writes the route only when the caller asked for one. Callers that want nothing
but the destination vertex pass no output list and pay nothing for the route reconstruction.
