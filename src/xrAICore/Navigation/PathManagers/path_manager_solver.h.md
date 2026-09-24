# src/xrAICore/Navigation/PathManagers/path_manager_solver.h

> Declares the policy that lets the path-search engine search the planner instead of a graph, implemented in [`path_manager_solver_inline.h`](path_manager_solver_inline.h.md).

**Needs** — [`path_manager_generic.h`](path_manager_generic.h.md) · [`../../Components/problem_solver.h`](../../Components/problem_solver.h.md) · [`path_manager_solver_inline.h`](path_manager_solver_inline.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_solver_inline.h`](path_manager_solver_inline.h.md)
**Tier floor** — T2: an adapter between two interfaces.

## Purpose

Declares the surface implemented in
[`path_manager_solver_inline.h`](path_manager_solver_inline.h.md). This is the file that makes
the goal/plan/action layer and the navigation layer one piece of machinery: it is selected
whenever the "graph" being searched is a planner.

## Exported units

- `setup(...)` — bind the planner, the workspace, the *edge* output, the endpoints and the limits
- `is_goal_reached(state)` — ask the planner
- `get_value(edge, reverse)` — the successor state, in the search's direction
- `edge(edge)` — the operator identifier, which is what the answer is made of
- `evaluate(from, to, edge)` / `estimate(state)` — the planner's edge cost and heuristic
- `init_path()` / `create_path(state)` — clear and write the plan
