# src/xrAICore/Navigation/graph_engine_inline.h

> Builds the three search workspaces and wraps each search entry point in the same four steps — bind a path manager, run, time it, report.

**Needs** — [`graph_engine.h`](graph_engine.h.md) · [`PathManagers/path_manager.h`](PathManagers/path_manager.h.md) · [`a_star_inline.h`](a_star_inline.h.md) · [`../AISpaceBase.hpp`](../AISpaceBase.hpp.md) · [Seam: Profiler](../../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)
**Used by** — [`graph_engine.h`](graph_engine.h.md)
**Tier floor** — T1: allocates and holds the search workspaces for the process lifetime.

## Purpose

Every search in the AI enters here. The file is short because there is almost nothing to it: the
work is in the search and in the path managers, and this layer only binds the two together and
measures the result. Its value to a rebuild is the *shape* of a search call, which is identical
across all five entry points.

## State

The engine owns exactly three things and holds them for the life of the process: the navigation search workspace, the planner search workspace, and the string search workspace. Each is a fully preallocated search object as described in [`a_star.h`](a_star.h.md). A path manager is stack-local per call and is not state.

## Construction

**Contract** — allocate all three workspaces up front and tune the navigation frontier's cost
ladder. Nothing is lazy: a graph engine that exists has paid its whole memory cost.

```text
FUNCTION construct(max_vertex_count)
  navigation <- AStar(max_vertex_count)
  navigation.frontier.set_min_bucket_value(0)
  navigation.frontier.set_max_bucket_value(2000)    # metres; the ladder's span
  planner <- AStar(16384)                           # see graph_engine.h on this number
  strings <- AStar(1024)
```

**Notes** — the ladder's upper bound of two thousand is a statement about level scale: no
navigation path in a shipped level is expected to cost more than two kilometres, and one that
does loses ordering resolution rather than failing. A rebuild targeting larger levels must raise
it or the frontier degenerates to a single bucket at the top.

## `search` — the common shape

**Contract** — run one search and return whether it reached the goal. The caller supplies the
graph, the start and goal vertices, a list to receive the path, and a parameter bundle carrying
the search budgets and whatever the specific path manager needs. The path list is written only on
success. Blocks for the duration of the search; the budget is what bounds that duration.

```text
FUNCTION search(graph, start, goal, out_path, parameters) -> bool
  manager <- PathManager(for this graph, parameters, and this workspace)
  manager.setup(graph, workspace.storage, out_path, start, goal, parameters)
  timer.begin()
  ok <- workspace.find(manager)
  timer.end()
  RETURN ok
```

**Invariants** — the path manager is a stack-local binding, not state: it holds references to the
graph, the workspace and the output list for exactly the duration of one search. The workspace
is what persists. One form of the entry point lets the caller pass its own path manager instead,
which is how a caller that needs to carry extra per-search state — a restrictor, a cover
preference — plugs it in without the engine knowing about it.

## The navigation entry point's validity check

**Contract** — before searching, the navigation form checks that both endpoints are valid
vertices of the *current level's* mesh, and refuses the search if not.

```text
FUNCTION search_navigation(graph, start, goal, out_path, parameters) -> bool
  IF NOT level_graph.valid_vertex_id(start) OR NOT level_graph.valid_vertex_id(goal)
    RETURN false                      # reported, then refused; never searched
  ... common shape ...
```

**Notes** — the check reaches the level mesh through the global AI-space handle rather than
through the graph argument, which means it is checking against the loaded level even when the
graph passed in is something else. It is a guard against the most common caller bug — a stale
vertex identity kept across a level change — and it is the reason an out-of-date identity
produces a refused search rather than an out-of-bounds read. A rebuild should validate against
the graph it was handed instead, which is strictly better and matches the intent.

Only this one form checks. The others trust their callers.

## `search` — the planner form

**Contract** — same shape, over the planner's workspace: the graph is a problem solver, the
endpoints are world states, and the output is a list of operator identifiers — the plan. The
result is whether a plan was found within the budget.

## `search` — the string form

**Contract** — same shape, over the string workspace: vertices are interned names. Used where the
graph's nodes are authored by name rather than numbered.

**Notes** — each entry point brackets its work in profiler zones and adds the elapsed time to the
engine's path timer, which the in-game statistics overlay reads. The profiler is optional (see
the seam) and a rebuild may drop both the zones and the timer without changing behaviour.
