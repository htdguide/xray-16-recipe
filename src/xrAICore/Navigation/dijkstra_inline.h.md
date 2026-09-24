# src/xrAICore/Navigation/dijkstra_inline.h

> The uniform-cost search loop — expand the cheapest open vertex, relax its neighbours, stop on the goal, the budget, or an exhausted frontier.

**Needs** — [`dijkstra.h`](dijkstra.h.md) · [`data_storage_constructor.h`](data_storage_constructor.h.md) · [`vertex_path.h`](vertex_path.h.md)
**Used by** — [`a_star_inline.h`](a_star_inline.h.md) · [`dijkstra.h`](dijkstra.h.md)
**Tier floor** — T2: it is pure graph algorithm over an abstract storage; nothing here touches a device or a byte layout. It is in the engine's manual tier only because the vertex records it walks are pooled for the frame budget.

## Purpose

This is the search half of the navigation engine. It knows nothing about navigation meshes,
creatures or costs: it asks a *path manager* for the neighbours of a vertex and the cost of
reaching each, and asks a *data storage* to keep the open frontier ordered. A* (see
[`a_star_inline.h`](a_star_inline.h.md)) is this loop with an added heuristic term, and inherits
the step function's skeleton rather than restating it.

The split into `initialize` / `step` / `find` is load-bearing: `step` advances the search by
exactly one vertex expansion, which is what makes the search *budgetable* — the caller counts
iterations and abandons a search that costs too much, rather than blocking the frame.

## State

```text
RECORD Search
  search_started : bool         # a search in progress; re-entering is a hard error
  storage        : DataStorage  # owns the open set, the closed set, the vertex pool
                                # and the parent links; see data_storage_constructor
```

**Invariants** — `search_started` is true exactly between `initialize` and `finalize`. A search
may not be started while another is running on the same engine: the vertex pool and the
visited-index table are single-occupancy, so a nested search would silently corrupt the outer
one. The check is not advisory — it is the only thing standing between a re-entrant caller and
a wrong path.

## `find`

**Contract** — run a complete search for the path manager's goal, from its start vertex. Returns
whether the goal was reached. Never blocks on anything but its own work, allocates nothing
during the search (the vertex pool is preallocated at construction), and writes its result
through the path manager, not through a return value. On every exit path — success, budget
exhausted, frontier empty — the path manager is finalized exactly once.

```text
FUNCTION find(path_manager) -> bool
  initialize(path_manager)
  iteration <- 0
  WHILE NOT storage.is_opened_empty()
    IF path_manager.is_limit_reached(iteration)      # budget; see Notes
      finalize(path_manager)
      RETURN false
    IF step(path_manager)
      finalize(path_manager)
      RETURN true
    iteration <- iteration + 1
  finalize(path_manager)
  RETURN false                                       # frontier exhausted: no path exists
```

**Notes** — the budget is checked *before* each expansion and never after the goal test, so a
search that reaches its goal on the very iteration that would have exceeded the budget still
succeeds. The budget itself lives in the path manager, not here: it is three separate ceilings
(cost of the cheapest open vertex, iteration count, number of distinct vertices visited), and
any one of them tripping ends the search. A failed search leaves the caller's path untouched —
`init_path` only runs on the success branch — so a caller that reuses a path buffer keeps the
previous path rather than getting an empty one.

## `initialize`

**Contract** — reset the storage and the path manager, create the start vertex with cost zero
and no parent, and place it in the open set. Fails hard if a search is already in progress.

```text
FUNCTION initialize(path_manager)
  FAIL WITH "recursive search" IF search_started
  search_started <- true
  storage.init()                    # new path generation; see vertex_manager_fixed
  path_manager.init()
  start <- storage.create_vertex(path_manager.start_node())
  start.f <- 0
  storage.assign_parent(start, none)
  storage.add_opened(start)
```

## `step`

**Contract** — expand one vertex: the cheapest one in the open set. Returns true when that
vertex is the goal, in which case the path has been handed to the path manager. Otherwise
relaxes each neighbour and returns false.

```text
FUNCTION step(path_manager) -> bool
  best <- storage.get_best()                  # minimum f in the open set
  IF path_manager.is_goal_reached(best.index)
    path_manager.init_path()
    path_manager.create_path(best)            # walks parent links back to the start
    RETURN true

  storage.add_best_closed()                   # mark it closed ...
  storage.remove_best_opened()                # ... and drop it from the frontier

  FOR EACH edge IN path_manager.neighbours_of(best.index)
    neighbour_index <- path_manager.get_value(edge)
    IF NOT path_manager.is_accessible(neighbour_index)
      CONTINUE                                # blocked vertex: never entered at all
    IF storage.is_visited(neighbour_index)
      neighbour <- storage.get_node(neighbour_index)
      IF storage.is_opened(neighbour)
        candidate <- best.f + path_manager.evaluate(best.index, neighbour_index, edge)
        IF neighbour.f > candidate
          old <- neighbour.f
          neighbour.f <- candidate
          storage.assign_parent(neighbour, best, path_manager.edge(edge))
          storage.decrease_opened(neighbour, old)   # re-place it in the ordering
      # else: closed, and with non-negative edge costs its cost is already final
    ELSE
      neighbour <- storage.create_vertex(neighbour_index)
      neighbour.f <- best.f + path_manager.evaluate(best.index, neighbour_index, edge)
      storage.assign_parent(neighbour, best, path_manager.edge(edge))
      storage.add_opened(neighbour)
  RETURN false
```

**Invariants** — a vertex is created exactly once per search generation; "visited" means
created, and it is either open or closed, never both. A closed vertex is never reopened: that
is only correct because every edge cost the engine supplies is non-negative, which is a
property of the cost functions in the path managers and not something checked here.

**Notes** — the closing order matters. The best vertex is added to the closed set *before* it is
removed from the open set, because the storage's `add_best_closed` reads the best entry out of
the frontier structure; reversing the two loses the vertex. Note also that the goal test happens
on *extraction*, not on insertion — with non-negative costs that is what makes the returned path
optimal, and it is why the goal vertex is expanded at zero further expense.

## `finalize`

**Contract** — tell the path manager the search is over and release the re-entrancy guard. Runs
on every exit path, including failure. Does not free the vertex pool: the pool is reused by the
next search and is only reset by `initialize`.
