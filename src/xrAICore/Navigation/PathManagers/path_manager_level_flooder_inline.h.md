# src/xrAICore/Navigation/PathManagers/path_manager_level_flooder_inline.h

> A flood fill of the navigation mesh — every reachable vertex within a radius of the start, delivered in order of increasing distance.

**Needs** — [`path_manager_level_flooder.h`](path_manager_level_flooder.h.md) · [`path_manager_params_flooder.h`](path_manager_params_flooder.h.md) · [`path_manager_level_inline.h`](path_manager_level_inline.h.md)
**Used by** — [`path_manager_level_flooder.h`](path_manager_level_flooder.h.md)
**Tier floor** — T2: integer cell arithmetic.

## Purpose

Several things the game layer does need not a route but a *neighbourhood*: which cells can a
creature reach from here, where could it take cover, which cells should a danger be spread
over. This policy answers all of those by running the ordinary mesh search with the goal
removed, so that it never terminates early and instead reports every vertex it settles.

It is the clearest illustration of the chapter's central reuse: nothing about the search engine
changed, only four of the policy's answers.

## State

```text
RECORD FloodState EXTENDS LevelSearchState
  start_x, start_z  : int    # cell coordinates of the flood centre
  radius_cells_sqr  : int    # the radius, converted once to squared cell units
  cell_size         : real
```

**Invariants** — the radius is converted from world units to cells at setup, once, by dividing
by the cell width and rounding to nearest, then squared. Every later test is integer.

## `setup`

**Contract** — as the mesh routing policy, plus: unpack the start vertex's cell coordinates, and
convert the request's range into a squared cell radius.

**Notes** — the request's *range* field is reinterpreted as a radius here. Its default of 6000
world units is far larger than any level, so a caller that does not supply one gets an
unbounded flood stopped only by the visited-vertex budget. Every real caller supplies one.

## `is_goal_reached`

**Contract** — appends the vertex to the output list, caches it as the currently-expanding
vertex, and always answers no.

**Invariants** — the output is therefore every settled vertex **in order of increasing cost from
the start**, because the driver settles cheapest-first. That ordering is part of the contract:
callers rely on the first entries being the nearest. The output list is required, not optional.

**Notes** — the search always ends by exhausting the frontier or by hitting a budget, and
therefore always reports failure. Failure here is the normal outcome and callers must ignore it
— the answer is the list, not the return value. A rebuild should give a flood its own entry
point that returns the list, rather than reporting a successful flood as a failed search.

## `estimate` / `evaluate`

**Contract** — zero, and one cell width. With no heuristic the driver is a uniform-cost
expansion, which is what makes the "increasing distance" ordering true.

## `is_accessible`

**Contract** — the mesh's own validity-and-access test, then the radius: the vertex's cell
coordinates must be within the squared cell radius of the start's. Circular in cell space, so
the flood is a disc rather than a square.

**Notes** — the radius is measured in *horizontal cell coordinates from the start cell*, not
along the path. A vertex twenty cells away by mesh distance but three cells away in a straight
line — across a wall, say — is inside the radius and will be flooded to if a route exists. The
radius bounds the *region*, the search bounds the *reachability*.

## `is_limit_reached`

**Contract** — the iteration and visited-vertex budgets only. The base policy's
estimated-total-cost test is dropped, because this policy has taken over that field's meaning.

## `create_path`

**Contract** — nothing. The output was written incrementally as vertices were settled; there is
no parent-link walk to do.
