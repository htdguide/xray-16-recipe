# src/xrAICore/Navigation/PathManagers/path_manager_generic_inline.h

> The default search policy — take every cost from the graph, use no heuristic, stop at the goal vertex, and honour the three budgets — which every other policy in this chapter refines rather than replaces.

**Needs** — [`path_manager_generic.h`](path_manager_generic.h.md) · [`path_manager_params.h`](path_manager_params.h.md)
**Used by** — [`path_manager_game_inline.h`](path_manager_game_inline.h.md) · [`path_manager_generic.h`](path_manager_generic.h.md) · [`path_manager_level_inline.h`](path_manager_level_inline.h.md) · [`path_manager_solver_inline.h`](path_manager_solver_inline.h.md)
**Tier floor** — T2: forwarding and three comparisons.

## Purpose

The base policy is the honest minimum: it knows only that there is a graph, a start, a goal
and three budgets. With a zero heuristic it turns the chapter's A* driver into a uniform-cost
search, which is the correct default because a heuristic is a claim about the *geometry* of a
graph and the base policy has no right to make one.

Everything specialised in this directory overrides two or three of these routines and inherits
the rest. Reading the overrides against this page is how to see what each specialisation
actually decides.

## State

```text
RECORD SearchPolicy
  graph                  : ref to Graph        # borrowed for the search's duration
  workspace              : ref to SearchStore  # the engine's open/closed structures
  output                 : optional<list<VertexId>>   # where the answer is written; may be absent
  start, goal            : VertexId
  max_range              : real
  max_iteration_count    : int
  max_visited_node_count : int
  current_vertex         : ref to VertexId     # the vertex whose edges are being walked
```

**Invariants** — the policy borrows everything and owns nothing; it is created per search and
discarded. The output list is optional throughout: a search issued purely for its side effects
(a flood fill that only wants the visited set, a reachability test) passes none, and every
routine that writes an answer checks first.

The "vertex whose edges are being walked" is remembered when the edge walk begins, because the
engine hands back only an edge afterwards and the graph needs both ends to resolve it. A
rebuild whose edge representation carries its source does not need this field.

## `setup`

**Contract** — binds the graph, the engine's workspace, the output list, the two endpoints and
the three budgets for one search. Does not touch the graph or the workspace. Allocates nothing.

## `evaluate`

**Contract** — the cost of one edge: asks the graph. The default defers entirely.

## `estimate`

**Contract** — zero. With no heuristic the driver behaves as a uniform-cost search, which is
always correct if not always fast.

## `is_metric_euclidian`

**Contract** — answers yes, unconditionally: a vertex once settled is never revisited even if a
cheaper route to it appears later.

**Invariants** — this answer is only sound when the heuristic never overestimates the remaining
cost. With the zero heuristic of the base policy it is trivially sound. **Every specialisation
that introduces a heuristic inherits this answer without restating it**, and at least one of
them introduces a heuristic that does overestimate — see
[`path_manager_level_inline.h`](path_manager_level_inline.h.md). Where that happens the search
returns a route that is valid but not necessarily shortest, deliberately.

**Notes** — the routine exists because the driver has a second, slower mode that does propagate
improvements into already-settled vertices. Nothing in this chapter ever selects it. A rebuild
may keep the option, but should name it for what it decides — "may settled vertices be
reopened" — rather than for the property of the metric that justifies the answer.

## `begin` / `get_value` / `edge`

**Contract** — `begin` remembers the vertex being expanded and asks the graph for its edge
range. `get_value` resolves an edge to the vertex on its far side, relative to that remembered
vertex. `edge` says what to record on the parent link; the default records the edge itself,
which is what a policy producing a vertex list needs.

## `is_accessible`

**Contract** — asks the graph whether a vertex may be entered.

## `is_goal_reached`

**Contract** — is this vertex the goal vertex. Identity, nothing more.

**Notes** — this routine is called on *every* expanded vertex in cheapest-first order, which is
why the goal-less policies in this directory hook it instead of overriding anything else: it is
the chapter's "visit" callback wearing a termination test's name.

## `is_limit_reached`

**Contract** — the search is out of budget when any of three things holds: the cheapest open
vertex's total estimated cost has reached the range limit, the expansion count has reached the
iteration limit, or the number of touched vertices has reached the visited limit.

**Invariants** — checked once per expansion, before the expansion. The first test reads the
*estimated total*, not the distance travelled; a policy that wants a true radius replaces this
routine rather than reinterpreting the number.

## `init_path` / `create_path` / `init` / `finalize`

**Contract** — `init_path` clears the output, called once the goal is found and before it is
written. `create_path` asks the engine's workspace to walk parent links from the found vertex
and write the route into the output. `init` and `finalize` are empty hooks the specialisations
use for per-search precomputation and for post-processing the answer.

**Notes** — the answer is written by the workspace rather than by the policy, because only the
workspace knows how parent links are stored. The policy decides *whether* to write it.
