# src/xrAICore/Navigation/PathManagers/path_manager_solver_inline.h

> The adapter that turns the path-search engine into a planner — the only search in the chapter whose answer is a list of *edges* rather than vertices, because a plan is a sequence of actions, not of world states.

**Needs** — [`path_manager_solver.h`](path_manager_solver.h.md) · [`../../Components/problem_solver_inline.h`](../../Components/problem_solver_inline.h.md) · [`path_manager_generic_inline.h`](path_manager_generic_inline.h.md) · [`../graph_engine_space.h`](../graph_engine_space.h.md)
**Used by** — [`path_manager_solver.h`](path_manager_solver.h.md)
**Tier floor** — T2: forwarding, plus one type narrowing.

## Purpose

Everything the planner needs from a search — a cost model, a heuristic, a termination test, a
successor relation — it already implements itself. This policy exists to connect the two
vocabularies, and in doing so makes the chapter's central claim concrete: *routing across a
level and choosing a sequence of actions are the same search with different answers to the same
questions.*

Its one genuine decision is what the answer is made of.

## State

```text
RECORD SolverSearchPolicy
  planner        : ref to Planner        # the "graph"
  workspace      : ref to SearchStore
  edge_output    : optional<list<operator_id>>   # the PLAN; not a list of states
  start, goal    : WorldState
  limits         : SearchLimits
```

**Invariants** — the output is a list of operator identifiers. Every other policy in this
directory writes vertices.

## `setup`

**Contract** — binds everything for one plan search. Does not call the base policy: the output
has a different element type, so the binding is written out rather than delegated.

**Notes** — the range limit is narrowed from the search engine's general distance type to the
planner's own narrower cost type on the way in. The planner's costs are small integers and its
limit is set to the saturated maximum of that narrower type, so the narrowing is lossless in
practice — but it is a silent truncation that a rebuild should make explicit, because a caller
passing a larger bound gets a smaller one without being told.

## `is_goal_reached` / `evaluate` / `estimate`

**Contract** — forwarded to the planner, which implements all three. The heuristic is the
planner's count of unsatisfied properties. See
[`../../Components/problem_solver_inline.h`](../../Components/problem_solver_inline.h.md) for
what those answers mean and for the admissibility caveat on the forward direction.

## `get_value`

**Contract** — the successor state, asking the planner for the direction — forward application
of an operator's effects, or backward regression of a goal — that the planner was configured
with at compile time.

**Notes** — the direction is passed as an argument whose default is the planner's own compile-time
answer, so the call site never supplies it. A rebuild should let the planner decide internally.

## `edge`

**Contract** — yields the operator identifier for an edge. This is the routine that makes the
answer a plan: it is what gets recorded on each parent link, so walking the links back produces
the actions rather than the states.

## `init_path` / `create_path`

**Contract** — clear the plan, then walk parent links from the found state collecting the
recorded operator identifiers. The collection is emitted in the order that matches the search's
direction: a forward search's links run from the goal back to the world and must be reversed to
be executable, a backward search's already run the right way.

**Notes** — there are two forms of the write-out, one taking an explicit workspace and order and
one using the bound workspace and the planner's compile-time direction. The two-argument form
has no caller in this chapter; it exists so a caller driving the search itself can choose. A
rebuild needs only the one.

Producing the plan in executable order is the contract the game layer depends on: it pops
actions off the front. Getting the reversal wrong yields a plan that is a valid sequence run
backwards, which fails in ways that look like an action bug rather than a search bug.
