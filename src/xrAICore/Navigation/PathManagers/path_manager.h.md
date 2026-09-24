# src/xrAICore/Navigation/PathManagers/path_manager.h

> The single name every search is issued through, and the dispatch rule that turns a (graph, parameter-set) pair into the right search policy.

**Needs** — [`path_manager_generic.h`](path_manager_generic.h.md) · [`path_manager_params.h`](path_manager_params.h.md) · [`path_manager_params_flooder.h`](path_manager_params_flooder.h.md) · [`path_manager_params_straight_line.h`](path_manager_params_straight_line.h.md) · [`path_manager_params_nearest_vertex.h`](path_manager_params_nearest_vertex.h.md) · [`path_manager_game.h`](path_manager_game.h.md) · [`path_manager_game_vertex.h`](path_manager_game_vertex.h.md) · [`path_manager_game_level.h`](path_manager_game_level.h.md) · [`path_manager_level.h`](path_manager_level.h.md) · [`path_manager_level_flooder.h`](path_manager_level_flooder.h.md) · [`path_manager_level_nearest_vertex.h`](path_manager_level_nearest_vertex.h.md) · [`path_manager_level_straight_line.h`](path_manager_level_straight_line.h.md) · [`path_manager_solver.h`](path_manager_solver.h.md)
**Used by** — [`graph_engine.h`](../graph_engine.h.md) · [`graph_engine_inline.h`](../graph_engine_inline.h.md)
**Tier floor** — T2: a dispatch table over search policies, resolved before the search runs.

## Purpose

There is one search algorithm in this chapter and half a dozen things people want out of it:
a route across a level, a route between levels, a route to *any* vertex on some level, a
flood fill of everything within a radius, the nearest reachable vertex to a point, the length
of a line-of-sight-shortened route, and a plan over actions. This file is the decision that
all of those are the *same* search with a different policy object, and it names the rule that
picks the policy.

## The dispatch rule

A path manager is selected by two things: **which graph** is being searched, and **which
parameter record** the caller passed. The parameter record is therefore not just a bag of
limits — it is the request, and choosing it is how a caller says what kind of answer it wants.

```text
(graph, parameters)                  -> policy
------------------------------------ ------------------------------------------
any graph, base parameters           -> the generic policy: uniform costs, no heuristic
game graph, base parameters          -> real inter-vertex distances, straight-line heuristic
game graph, game-level parameters    -> stop at any vertex on a named level
game graph, game-vertex parameters   -> confine the search to vertices of wanted terrain kinds
level graph, base parameters         -> grid costs, weighted Manhattan heuristic
level graph, flooder parameters      -> no goal; collect every vertex within a radius
level graph, nearest-vertex params   -> no goal; keep the one vertex closest to a point
level graph, straight-line params    -> route, then measure its line-of-sight-shortened length
planner, base parameters             -> the plan search: vertices are world states, edges actions
```

**Notes** — the selection happens before the search starts and costs nothing at search time;
every policy routine is resolved statically. A rebuild in a language without compile-time
specialization can make the policy an interface the caller supplies and pay one indirect call
per routine — the routines are called once per expanded vertex and once per edge, so the cost
is real but bounded, and the structure is unchanged.

Two policies are built only for the *offline* tool that compiles navigation data, and two only
for the running game. The straight-line policy is the tool's; the nearest-vertex policy and
the plan policy are the game's. That split is a build-configuration fact, not a design one — a
rebuild may ship all of them everywhere.
