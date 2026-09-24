# src/xrAICore/Navigation/PathManagers/path_manager_level_flooder.h

> Declares the goal-less mesh search that collects every reachable vertex within a radius, implemented in [`path_manager_level_flooder_inline.h`](path_manager_level_flooder_inline.h.md).

**Needs** — [`path_manager_level.h`](path_manager_level.h.md) · [`path_manager_params_flooder.h`](path_manager_params_flooder.h.md) · [`path_manager_level_flooder_inline.h`](path_manager_level_flooder_inline.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_level_flooder_inline.h`](path_manager_level_flooder_inline.h.md)
**Tier floor** — T2: a radius test in integer cell units.

## Purpose

Declares the surface implemented in
[`path_manager_level_flooder_inline.h`](path_manager_level_flooder_inline.h.md). It refines the
mesh routing policy into a flood fill by removing the goal and the heuristic and adding a
radius.

## Exported units

- `setup(...)` — as the mesh policy, plus the start's cell coordinates and the radius in cells
- `is_goal_reached(vertex)` — never true; records the vertex instead
- `evaluate(from, to, edge)` — one cell width
- `estimate(vertex)` — zero
- `is_accessible(vertex)` — the mesh's own test, then the radius
- `is_limit_reached(iterations)` — the iteration and visited budgets only
- `create_path(vertex)` — nothing; the answer was already written
