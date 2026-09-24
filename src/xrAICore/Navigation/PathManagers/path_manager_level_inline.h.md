# src/xrAICore/Navigation/PathManagers/path_manager_level_inline.h

> Routing on the level's navigation mesh: every step costs one cell, the heuristic is twice the Manhattan distance to the goal, and settled vertices are never revisited — a deliberately inflated search that is fast and is not guaranteed to be shortest.

**Needs** — [`path_manager_level.h`](path_manager_level.h.md) · [`../level_graph.h`](../level_graph.h.md) · [`path_manager_generic_inline.h`](path_manager_generic_inline.h.md)
**Used by** — [`path_manager_level.h`](path_manager_level.h.md) · [`path_manager_level_flooder_inline.h`](path_manager_level_flooder_inline.h.md) · [`path_manager_level_nearest_vertex_inline.h`](path_manager_level_nearest_vertex_inline.h.md) · [`path_manager_level_straight_line_inline.h`](path_manager_level_straight_line_inline.h.md)
**Tier floor** — T2: integer arithmetic on cell coordinates.

## Purpose

This is the cost model of walking around a level, and it rests entirely on one property of the
navigation mesh: **it is a uniform grid**. Every vertex occupies one cell of fixed width, every
vertex has at most four neighbours, and neighbours are adjacent cells. Once that is granted,
distances become cell counts and the whole cost model collapses to integer arithmetic on
packed cell coordinates.

## State

```text
RECORD LevelSearchState
  cell_size          : real    # the mesh's cell width, in world units; from the mesh header
  cell_size_squared  : real
  start_x, start_z   : int     # cell coordinates, fixed at search start
  goal_x, goal_z     : int     # cell coordinates, fixed at search start
  cur_x, cur_z       : int     # cell coordinates of the vertex currently being expanded
  current_vertex     : ref to MeshVertex   # that same vertex, resolved once
```

**Invariants** — `cur_x`/`cur_z` and `current_vertex` are refreshed by `is_goal_reached` and
consumed by `estimate`, `begin` and `get_value`. The driver calls them in that order for every
expansion, so the coupling holds — but it is a coupling, and a rebuild that reorders the
driver's calls breaks this policy silently rather than loudly.

## `setup` / `init`

**Contract** — `setup` caches the mesh's cell size and its square. `init`, run once the search
has begun, unpacks the start and goal vertices into cell coordinates and seeds the
current-expansion coordinates with the start's.

**Notes** — cell coordinates come from unpacking a single packed integer stored on the vertex,
not from its world position. That is what keeps the heuristic to integer subtraction.

## `evaluate`

**Contract** — one cell width, for every edge, always.

**Invariants** — sound because the mesh is 4-connected on a uniform grid: an edge always joins
horizontally adjacent cells.

**Notes** — the **vertical** component is ignored. Climbing a staircase costs the same as
crossing flat ground, and a route that gains a lot of height is indistinguishable from a level
one of the same cell count. This is a deliberate simplification — the file retains a disabled
variant that scaled the height difference and folded it into the step cost — and its
consequence is that creatures do not prefer flat routes. A rebuild that adds height back must
also raise the heuristic's admissibility argument, because the heuristic below assumes each
step costs exactly one cell.

## `estimate`

**Contract** — twice the cell size, times the Manhattan distance in cells from the
currently-expanding vertex to the goal.

```text
FUNCTION estimate() -> real
  RETURN 2 * cell_size * (abs(goal_x - cur_x) + abs(goal_z - cur_z))
```

**Invariants** — the true remaining cost is at least `cell_size × Manhattan`, because each step
changes exactly one cell coordinate by one and costs one cell. The estimate is therefore **about
twice the admissible value**: this is a weighted A*, and the weight is 2.

**Notes** — two things about this routine are load-bearing and neither is obvious.

*The factor of two is the design.* It biases the search hard toward the goal and cuts the number
of expanded vertices dramatically on the open, mostly-obstacle-free meshes the game ships. The
price is stated plainly: combined with the base policy's refusal to re-open settled vertices
(inherited unchanged from
[`path_manager_generic_inline.h`](path_manager_generic_inline.h.md)), **routes on the level mesh
are valid but not necessarily shortest**, and can be up to twice the optimal length in the worst
case. Every creature in the game walks such a route. A rebuild that drops the factor gets shorter
paths and a slower, more memory-hungry search; the original chose speed, and the shipped level
meshes are forgiving enough that the difference is rarely visible.

*The estimate ignores the vertex it is asked about* and uses the coordinates of the vertex
currently being expanded. Since a neighbour is always one cell away, the two differ by at most
one cell's worth — but the heuristic is thereby a property of the expansion rather than of the
vertex, which means a vertex reached from two directions is estimated differently. A rebuild
should compute it from the vertex; the result is marginally better and strictly simpler.

The file also carries disabled variants of both the cost and the estimate — a Euclidean form
with a height term, and an octile form for eight-connected movement. They are the shape of the
experiments that produced these numbers and are worth reading before changing them.

## `is_goal_reached`

**Contract** — answers whether this vertex is the goal, and on the way refreshes the
currently-expanding vertex pointer and its cell coordinates.

**Notes** — the refresh is a side effect in a predicate, and it is how the rest of this policy
gets its working state. It is the clearest case in the chapter of an interface shaped by the
driver's call order rather than by meaning; a rebuild should give the driver an explicit
"expanding this vertex" hook and leave the predicate pure.

## `is_accessible`

**Contract** — defers to the mesh, which answers yes only for a valid vertex identifier whose
access bit is set. The access mask is a per-vertex runtime overlay — it is how the game closes
off parts of a level without editing the compiled mesh.

## `begin` / `get_value`

**Contract** — walk the cached vertex's four neighbour slots and resolve each to the vertex it
links to. Driven from the cached vertex rather than from a vertex identifier, which saves an
identifier-to-vertex resolution per expansion — worth doing here because this is the hottest
loop in the chapter.
