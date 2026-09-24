# src/xrAICore/Navigation/a_star_inline.h

> The heuristic search — the uniform-cost loop with an estimate-to-goal added to each vertex's ordering key, and the one branch that decides whether a closed vertex may ever be improved.

**Needs** — [`a_star.h`](a_star.h.md) · [`dijkstra_inline.h`](dijkstra_inline.h.md)
**Used by** — [`a_star.h`](a_star.h.md) · [`graph_engine_inline.h`](graph_engine_inline.h.md)
**Tier floor** — T2: graph algorithm over abstract storage; no device, no byte layout.

## Purpose

Every path the engine finds — through a level's navigation mesh, across the cross-level graph,
and through the space of world states in the planner — is found by this loop. It differs from
uniform-cost search in exactly two places: each vertex carries a split cost, and a vertex
already closed may be revisited when the heuristic is not trusted.

## State

Adds two fields to the search record of each vertex:

```text
RECORD AStarVertexData EXTENDS DijkstraVertexData
  g : Distance      # cost actually paid from the start to this vertex
  h : Distance      # estimated remaining cost from this vertex to the goal
  # invariant: f == g + h at all times; f is what the frontier orders by
```

**Invariants** — `h` is computed once, when the vertex is first created, and never recomputed;
it depends only on the vertex and the goal, both of which are fixed for one search. `g` may fall
as better paths are found, and `f` must be re-derived from `g + h` on every such fall, in that
order, before the frontier is told the key changed.

## `find`

**Contract** — identical in shape to the uniform-cost `find`: initialize, then expand one vertex
per iteration until the goal is reached, the budget is exhausted, or the frontier empties.
Returns whether the goal was reached. Finalizes the path manager on every exit.

```text
FUNCTION find(path_manager) -> bool
  initialize(path_manager)
  iteration <- 0
  WHILE NOT storage.is_opened_empty()
    IF path_manager.is_limit_reached(iteration)
      finalize(path_manager); RETURN false
    IF step(path_manager)
      finalize(path_manager); RETURN true
    iteration <- iteration + 1
  finalize(path_manager)
  RETURN false
```

## `initialize`

**Contract** — as for uniform cost, except the start vertex is given a real estimate rather than
a zero key, so the very first expansion is already aimed at the goal.

```text
FUNCTION initialize(path_manager)
  FAIL WITH "recursive search" IF search_started
  search_started <- true
  storage.init()
  path_manager.init()
  start <- storage.create_vertex(path_manager.start_node())
  start.g <- 0
  start.h <- path_manager.estimate(start.index)
  start.f <- start.g + start.h
  storage.assign_parent(start, none)
  storage.add_opened(start)
```

## `step`

**Contract** — expand the vertex with the smallest `f`. Returns true if that vertex is the goal,
having handed the path to the path manager. Otherwise relaxes every accessible neighbour and
returns false.

```text
FUNCTION step(path_manager) -> bool
  best <- storage.get_best()
  IF path_manager.is_goal_reached(best.index)
    path_manager.init_path()
    path_manager.create_path(best)
    RETURN true
  storage.add_best_closed()
  storage.remove_best_opened()

  FOR EACH edge IN path_manager.neighbours_of(best.index)
    n_index <- path_manager.get_value(edge)
    IF NOT path_manager.is_accessible(n_index)
      CONTINUE
    IF NOT storage.is_visited(n_index)
      n <- storage.create_vertex(n_index)
      n.g <- best.g + path_manager.evaluate(best.index, n_index, edge)
      n.h <- path_manager.estimate(n.index)        # computed once, here
      n.f <- n.g + n.h
      storage.assign_parent(n, best, path_manager.edge(edge))
      storage.add_opened(n)
      CONTINUE
    n <- storage.get_node(n_index)
    IF storage.is_opened(n)
      relax_open(n, best, edge)
    ELSE IF NOT path_manager.is_metric_euclidian()
      relax_closed(n, best, edge)
    # else: closed under a trusted heuristic; its cost is already final
  RETURN false
```

## `relax_open` — improving a vertex still in the frontier

```text
STEP relax_open(n, best, edge)
  candidate <- best.g + path_manager.evaluate(best.index, n.index, edge)
  IF n.g <= candidate
    RETURN                                   # the known path is as good or better
  old_key <- n.f
  n.g <- candidate
  n.f <- n.g + n.h                           # h is unchanged; only g moved
  storage.assign_parent(n, best, path_manager.edge(edge))
  storage.decrease_opened(n, old_key)        # frontier re-places it under the new key
```

**Notes** — the old key is handed to the frontier because one of the two frontier
implementations needs to find the entry under its previous ordering before moving it. A rebuild
whose priority structure supports decrease-key directly can drop the argument.

## `relax_closed` — the branch that depends on the heuristic

**Contract** — only reachable when the path manager declares its metric non-euclidian. A closed
vertex whose cost improves is corrected, its parent re-pointed, and *all of its successors* told
to recompute — because they were built on a cost that has just changed.

```text
STEP relax_closed(n, best, edge)
  candidate <- best.g + path_manager.evaluate(best.index, n.index, edge)
  IF n.g <= candidate
    RETURN
  n.g <- candidate
  n.f <- n.g + n.h
  storage.assign_parent(n, best, path_manager.edge(edge))
  storage.update_successors(n)
```

**Notes** — this is the file's one real decision, and it is worth stating as a guarantee rather
than as code. When the estimate is a true lower bound on the remaining cost — which it is when
it is the straight-line distance to the goal under a metric where travel cost is at least
distance — a vertex removed from the frontier already has its final cost, and touching closed
vertices is wasted work. The path manager asserts that property by declaring its metric
euclidian, and the search then *guarantees* an optimal path. When the metric is not euclidian
the search still runs and still returns a path, but it only *attempts* optimality: reaching the
goal no longer proves the path is the cheapest one, and the code says so in as many words.

In the shipped engine every path manager declares its metric euclidian, so the closed branch is
dead and the successor-update path is a hard error if ever reached (see
[`vertex_path_inline.h`](vertex_path_inline.h.md)). A rebuild may omit the branch entirely and
lose nothing that ships — but it then owes the same promise: no cost function that
underestimates travel cost may be plugged in, or the paths quietly stop being shortest.

## `Search` construction and teardown

**Contract** — constructed with the maximum number of vertices one search may visit; that number
sizes the vertex pool and the visited-set table up front, so no allocation happens during a
search. Nothing else is configured here — the frontier's bucket range is set by the caller after
construction.
