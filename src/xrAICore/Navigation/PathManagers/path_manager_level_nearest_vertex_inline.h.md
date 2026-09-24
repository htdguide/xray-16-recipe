# src/xrAICore/Navigation/PathManagers/path_manager_level_nearest_vertex_inline.h

> Flood the mesh within a radius and keep the single reachable vertex that ends up closest, horizontally, to a target position that need not be on the mesh at all.

**Needs** — [`path_manager_level_nearest_vertex.h`](path_manager_level_nearest_vertex.h.md) · [`path_manager_params_nearest_vertex.h`](path_manager_params_nearest_vertex.h.md) · [`path_manager_level_inline.h`](path_manager_level_inline.h.md)
**Used by** — [`path_manager_level_nearest_vertex.h`](path_manager_level_nearest_vertex.h.md)
**Tier floor** — T2: a squared-distance comparison.

## Purpose

A creature is told to go to a position, and the position is not a mesh vertex: it is under a
ledge, inside geometry, on the far side of a gap, or simply a number a script produced. Asking
"which mesh vertex is nearest that point" by geometry alone gives an answer on the wrong side
of a wall. This policy asks the question the creature actually means — *which vertex that I can
walk to is nearest* — by flooding outward from where it stands and keeping a running minimum.

## State

```text
RECORD NearestVertexState EXTENDS LevelSearchState
  start_x, start_z       : int    # cell coordinates of the flood centre
  radius_cells_sqr       : int
  cell_size              : real
  target_position        : (real, real, real)
  best_squared_distance  : real   # running minimum; starts at the largest representable value
```

**Invariants** — the output list holds at most one vertex at any time: the current best. It is
cleared at setup and rewritten on every improvement.

## `setup`

**Contract** — as the mesh routing policy, plus: the start's cell coordinates, the radius
converted to squared cell units, the target position copied, the running minimum seeded to
infinity, and the output list emptied. The output list is required.

## `is_goal_reached`

**Contract** — measures the **horizontal** squared distance from the settled vertex's world
position to the target, and if it improves on the running minimum, replaces both the minimum
and the single-element output. Always answers no.

**Invariants** — when the search ends the output holds exactly the best vertex found, or is
empty if nothing was reachable within the radius.

**Notes** — distance is measured horizontally, ignoring height. That is right for the question:
a vertex directly below a target on a balcony is a good place to stand, and a vertex on the
same floor but around a corner is not made better by being level with it. It does mean that on
multi-storey geometry the answer can be on the wrong floor, and callers that care must filter
afterwards.

Squared distance is compared throughout; no square root is taken. The comparison is all that is
needed and the absolute value is never reported.

The search, like the flood it is built on, always ends in reported failure. The answer is the
output list.

## `evaluate` / `estimate` / `is_accessible` / `is_limit_reached` / `create_path`

**Contract** — identical to the flood fill: one cell width per step, no heuristic, the mesh's
own accessibility test followed by the circular cell radius, the iteration and visited budgets
without the estimated-cost test, and no parent-link walk. See
[`path_manager_level_flooder_inline.h`](path_manager_level_flooder_inline.h.md), whose notes
apply unchanged.

**Notes** — the duplication between this policy and the flood fill is real: they differ in one
routine. A rebuild should express the flood once and parameterise what it does with each settled
vertex.
