# src/xrAICore/Navigation/PathManagers/path_manager_game_inline.h

> Routing across the coarse cross-level graph — true stored edge lengths, a straight-line heuristic, no budget at all.

**Needs** — [`path_manager_game.h`](path_manager_game.h.md) · [`../game_graph.h`](../game_graph.h.md) · [`path_manager_generic_inline.h`](path_manager_generic_inline.h.md)
**Used by** — [`path_manager_game.h`](path_manager_game.h.md) · [`path_manager_game_level_inline.h`](path_manager_game_level_inline.h.md) · [`path_manager_game_vertex_inline.h`](path_manager_game_vertex_inline.h.md)
**Tier floor** — T2: distances and a comparison.

## Purpose

The cross-level graph is small — a few thousand vertices for an entire game — and metric: every
vertex has a world position and every edge carries the real distance between its ends. That
combination makes the textbook A* configuration available, and this policy takes it.

## `setup`

**Contract** — as the base policy, and additionally resolves the goal vertex once and keeps it.
The heuristic is evaluated for every vertex the search touches and would otherwise repeat that
lookup every time.

## `evaluate`

**Contract** — the distance recorded on the edge. Not recomputed from the endpoints' positions:
the compiled graph's edge lengths are authoritative and may reflect a route that is not a
straight line.

## `estimate`

**Contract** — the straight-line distance from the vertex's world position to the goal's.

**Invariants** — this never exceeds the true remaining cost, because every edge's stored length
is at least the straight-line distance between its ends and distances add along a route. The
heuristic is therefore admissible, the base policy's refusal to re-open settled vertices is
sound, and **routes on the cross-level graph are genuinely shortest**. This is the only search
in the chapter of which that is true.

**Notes** — the invariant is a claim about the *data*, not about the code: it holds because the
graph compiler writes real distances. A rebuild that recomputes or approximates edge lengths
must keep them at or above the straight-line distance or silently lose optimality.

## `is_limit_reached`

**Contract** — never. This search runs to completion.

**Notes** — deliberate and affordable: the graph is small, the heuristic is good, and the caller
is the off-screen simulation, which is not on the frame's critical path. It is also the reason
the budgets in the request record are meaningless for this policy — a caller that fills them
in is ignored.

## `is_accessible`

**Contract** — the graph's own per-vertex enable flag, which the game layer toggles to close
routes temporarily (a blocked passage, a level not yet unlocked). Distinct from the base
policy's validity check: a vertex can exist and be disabled.
