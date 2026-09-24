# src/xrAICore/Navigation/PathManagers/path_manager_params.h

> The three budgets every search carries, and the meaning of each — the record every other parameter set extends.

**Needs** — _(none)_
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_generic_inline.h`](path_manager_generic_inline.h.md) · [`path_manager_level.h`](path_manager_level.h.md) · [`path_manager_params_flooder.h`](path_manager_params_flooder.h.md) · [`path_manager_params_game_level.h`](path_manager_params_game_level.h.md) · [`path_manager_params_game_vertex.h`](path_manager_params_game_vertex.h.md) · [`path_manager_params_nearest_vertex.h`](path_manager_params_nearest_vertex.h.md) · [`path_manager_params_straight_line.h`](path_manager_params_straight_line.h.md)
**Tier floor** — T2: three numbers and their defaults.

## Purpose

Every search in this chapter is bounded, and this record is where the bounds live. It is also
the base of every specialised request, so its fields are present in all of them; a caller that
wants a plain route from A to B passes nothing but this.

## State

```text
RECORD SearchLimits
  max_range               : real   # stop when the cheapest open vertex's total estimated
                                   #   cost reaches this. NOT a radius: it bounds f = g + h.
                                   #   Default: the largest representable value = no limit.
  max_iteration_count     : int    # stop after this many expanded vertices.
                                   #   Default: the largest representable value = no limit.
  max_visited_node_count  : int    # stop when this many vertices have been touched.
                                   #   Default in the running game: 65500.
```

**Invariants** — the visited-vertex budget must stay below the capacity of the search engine's
fixed vertex pool, which the game configures at 65536 vertices. 65500 is that capacity minus a
small margin, and it is the reason the number is not round. The offline navigation-data
compiler runs with a far larger pool and sets this budget to "no limit" instead.

**Notes** — `max_range` bounding *estimated total cost* rather than distance from the start is
the subtle one, and it means the limit interacts with the heuristic: a policy with an inflated
heuristic reaches the range limit sooner than its true path length would suggest. The policies
that want a genuine radius — the flooder and the nearest-vertex search — reinterpret this field
as one and replace the limit test accordingly.

A search that hits any of the three limits reports failure. The search is **not resumable**:
there is no saved frontier and no continuation. A caller that wants to spread work across
frames issues a bounded search each frame and starts over, which is why the budgets are small
enough to be affordable every frame rather than large enough to guarantee an answer.

The record also carries a trivially-true "are these parameters still current" query, so that
policies whose request can go stale can answer it uniformly. Nothing in this chapter overrides
it.
