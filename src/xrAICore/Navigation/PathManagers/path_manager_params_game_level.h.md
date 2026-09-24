# src/xrAICore/Navigation/PathManagers/path_manager_params_game_level.h

> The request for "get me to any vertex on that level" — a level identifier in, the vertex that was reached out.

**Needs** — [`path_manager_params.h`](path_manager_params.h.md)
**Used by** — [`path_manager_game_level.h`](path_manager_game_level.h.md) · [`path_manager_game_level_inline.h`](path_manager_game_level_inline.h.md)
**Tier floor** — T2: a request record.

## Purpose

The coarse cross-level graph is how the off-screen simulation moves entities between levels.
An entity ordered to another level does not have a destination vertex there — it has a
destination *level*. This request expresses that: search the cross-level graph until any vertex
on the named level is reached, and report which one it was.

## State

```text
RECORD GameLevelRequest EXTENDS SearchLimits
  level_id               : int                 # the level to reach
  vertex_id              : int                 # OUT: the first vertex on that level the
                                               #   search reached; set to the invalid marker
                                               #   at search start
  path                   : optional<list<int>> # OUT: the route, when one was asked for
  max_range              : real  # default 6000
  max_iteration_count    : int   # default: no limit
  max_visited_node_count : int   # default: no limit
```

**Invariants** — the output vertex is reset to the invalid marker when the search is set up, so
a failed search leaves it invalid rather than stale. The caller must check it rather than
trusting the search's success answer alone.

**Notes** — "the first vertex on that level the search reached" is well defined only because the
search that uses this request runs with a zero heuristic — it is a uniform-cost search, so the
first vertex on the wanted level to be expanded is the cheapest one to reach. With a heuristic
it would merely be *a* vertex on that level. See
[`path_manager_game_level_inline.h`](path_manager_game_level_inline.h.md).
