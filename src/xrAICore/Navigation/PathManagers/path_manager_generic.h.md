# src/xrAICore/Navigation/PathManagers/path_manager_generic.h

> Declares the search policy every specialisation refines — the complete list of questions the search engine asks about a search — implemented in [`path_manager_generic_inline.h`](path_manager_generic_inline.h.md).

**Needs** — [`path_manager_generic_inline.h`](path_manager_generic_inline.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_game.h`](path_manager_game.h.md) · [`path_manager_generic_inline.h`](path_manager_generic_inline.h.md) · [`path_manager_level.h`](path_manager_level.h.md) · [`path_manager_solver.h`](path_manager_solver.h.md)
**Tier floor** — T2: an interface plus defaults that forward to a graph.

## Purpose

Declares the surface implemented in
[`path_manager_generic_inline.h`](path_manager_generic_inline.h.md). Reading this list is the
fastest way to understand what the search engine in this chapter needs from a caller, because
a policy must answer every one of these and nothing else.

## Exported units

The search's subject:

- `setup(graph, workspace, output, start, goal, limits)` — bind everything for one search
- `init()` — a hook run once the search has begun, before the first vertex is expanded
- `start_node()` / `goal_node()` — the endpoints

The cost model:

- `evaluate(from, to, edge)` — the cost of traversing one edge
- `estimate(vertex)` — the heuristic: estimated remaining cost from a vertex
- `is_metric_euclidian()` — may a vertex already settled be re-opened when a cheaper route to
  it is found? Answering yes forbids it.

Neighbours:

- `begin(vertex, first, last)` — the outgoing edges of a vertex
- `get_value(edge)` — the vertex an edge leads to
- `edge(edge)` — what to record on the parent link so the answer can name edges, not vertices
- `is_accessible(vertex)` — may the search enter this vertex at all

Termination:

- `is_goal_reached(vertex)` — is this the answer; also the hook a goal-less search uses to
  observe every expanded vertex
- `is_limit_reached(iteration_count)` — has the search run out of budget

The answer:

- `init_path()` — clear the output before writing it
- `create_path(vertex)` — walk parent links from the found vertex and write the answer out
- `finalize()` — a hook run when the search ends, successfully or not
