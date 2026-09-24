# src/xrAICore/Components/problem_solver_inline.h

> The planner: it holds the actions and the property evaluators, decides when the standing plan has gone stale, and presents itself to the path-search engine as a graph whose vertices are world states and whose edges are actions.

**Needs** — [`problem_solver.h`](problem_solver.h.md) · [`operator_abstract_inline.h`](operator_abstract_inline.h.md) · [`condition_state_inline.h`](condition_state_inline.h.md) · [`../Navigation/graph_engine.h`](../Navigation/graph_engine.h.md) · [`../Navigation/graph_engine_space.h`](../Navigation/graph_engine_space.h.md) · [`../../Include/xrAPI/xrAPI.h`](../../Include/xrAPI/xrAPI.h.md)
**Used by** — [`operator_abstract_inline.h`](operator_abstract_inline.h.md) · [`problem_solver.h`](problem_solver.h.md) · [`path_manager_solver_inline.h`](../Navigation/PathManagers/path_manager_solver_inline.h.md)
**Tier floor** — T2: a symbolic search with a fixed visited-state budget; nothing device- or format-facing.

## Purpose

This is the goal/plan/action layer's engine room. A caller declares a goal as a partial world
state, registers a set of actions and a set of property evaluators, and calls `solve`. It gets
back a sequence of action identifiers that, executed in order, is believed to bring the world
to the goal.

Two claims must be kept apart, because the game layer depends on both and they are not the same
strength:

- **Guaranteed.** Every action in the returned plan had its preconditions satisfied at the
  point in the plan where it appears, *given the world as measured during the search*. The plan
  is never longer than the visited-state budget allows. The plan is recomputed whenever any
  property the previous plan relied on has changed value, so a stale plan is never executed.
- **Merely attempted.** That the plan is the cheapest one. The forward search's heuristic can
  overestimate (see `estimate_edge_weight`), so the search may settle for a costlier plan than
  exists. That a plan exists at all: the search gives up after a bounded number of visited
  states and reports failure, and a failure is indistinguishable from "no plan exists". That
  the plan still works when it is executed: the world is re-measured every frame and the plan
  is discarded the moment it disagrees, which is the re-planning loop rather than a guarantee
  about any one plan.

## State

```text
RECORD Planner
  operators      : list<(operator_id, Operator)>   # ascending by operator_id; owned
  evaluators     : map<property_id, Evaluator>     # ordered; owned
  solution       : list<operator_id>               # the standing plan
  target_state   : WorldState                      # the goal, partial
  current_state  : WorldState                      # the MEASURED world: only properties some
                                                   #   search has actually asked about
  temp           : WorldState                      # scratch successor, reused per edge
  applied        : bool      # did the last edge expansion yield a real successor
  actuality      : bool      # cleared by any edit to operators, evaluators or goal
  solution_changed : bool    # did the last solve produce a different plan
  failed         : bool      # did the last solve fail to find one
```

**Invariants**

- Operators are kept ascending by identifier so lookup and insertion are binary searches, and
  so the edge order out of a state is stable — the search's tie-breaking depends on it.
- Every property named by any operator's preconditions or effects must have a registered
  evaluator. Checked on operator registration in debug builds only; without it, the search
  reaches a property it cannot measure.
- `current_state` is a *cache*, not a snapshot. It grows during a search as properties are
  measured and is cleared at the start of each re-plan. A property's value is measured at most
  once per plan.
- A search vertex is a **delta against `current_state`** — see
  [`operator_abstract_inline.h`](operator_abstract_inline.h.md). The start vertex of a forward
  search is therefore the empty state.

## `solve`

**Contract** — re-plans if and only if the standing plan is stale. Reports through
`solution_changed` whether the plan differs and through `failed` whether planning succeeded.
Runs the search synchronously: it is not resumable across frames, unlike the navigation
searches in this chapter. Allocates within a fixed pool owned by the search engine.

```text
FUNCTION solve()
  solution_changed <- false
  IF actual() THEN RETURN                  # standing plan is still good; do nothing

  actuality        <- true
  solution_changed <- true
  current_state.clear()                    # forget every measured property; re-measure lazily

  IF forward
    start <- current_state (now empty)     # the world as it is
    goal  <- target_state
  ELSE
    start <- target_state                  # the goal, to be regressed
    goal  <- current_state
  failed <- NOT graph_engine.search(self, start, goal, INTO solution,
                                    limits: range = unbounded,
                                            iterations = unbounded,
                                            visited states <= 8000)
```

**Notes** — the two directions swap the endpoints and nothing else; the direction-dependent
behaviour lives in `is_goal_reached` and `estimate_edge_weight`.

Only the visited-state count is actually bounded. The distance and iteration limits are set to
the maximum representable value of their types, which is the idiom this codebase uses for "no
limit"; a rebuild should use an explicit absent-limit instead of a saturated integer.

**8000 is not arbitrary**: the search engine's solver configuration allocates a fixed pool of
8192 vertices and never grows it, so the budget must sit below that or the pool overruns. The
margin is 192 states. See [`../Navigation/graph_engine.h`](../Navigation/graph_engine.h.md).

The search reaches its engine through the global environment struct rather than being handed
one — a service-locator reach that a rebuild should turn into an injected dependency.

## `actual`

**Contract** — is the standing plan still valid? Two independent tests. First, the staleness
flag, which any edit to the operator set, the evaluator set or the goal clears. Second, and
more importantly: every property that was measured during the last plan is measured *again*,
and if any answers differently the plan is stale. Returns true only when nothing moved.

```text
FUNCTION actual() -> bool
  IF NOT actuality THEN RETURN false
  FOR EACH (property, cached_value) IN current_state
    IF evaluators[property].evaluate() != cached_value THEN RETURN false
  RETURN true
```

**Invariants** — the walk relies on both sequences being ordered by property identifier, so the
evaluator cursor only moves forward; it binary-searches ahead when it falls behind rather than
restarting. Every measured property must have an evaluator, asserted here.

**Notes** — this is the re-planning trigger and it is exactly the right one: only properties the
*previous plan actually consulted* are re-measured, so a world change nobody's plan depended on
costs nothing. It is also the per-frame cost of the whole layer — one evaluator call per
property the current plan touches, every time the owner ticks.

## `evaluate_condition`

**Contract** — measures one property through its registered evaluator and inserts the result
into the measured world at the position the caller is standing on, then hands the caller back
its cursors. Idempotent in effect: a property is only ever measured through here when it is
absent.

**Notes** — the cursor hand-back is the incidental C++: inserting into the measured world may
move it, invalidating positions held by the merge walks in
[`operator_abstract_inline.h`](operator_abstract_inline.h.md). A rebuild with stable positions
deletes the hand-back; one without it must keep the insertion point in step.

The routine is declared read-only on the planner but mutates the measured world — the cache is
deliberately excluded from the planner's logical state. A rebuild should model the measured
world as an explicitly mutable cache rather than hide the mutation.

## The graph face

These are what the search engine calls. Together they say: *the neighbours of a state are all
operators, filtered after the fact.*

### `begin`

**Contract** — yields the whole operator list as the outgoing edges of any state, ignoring the
state entirely.

**Notes** — the branching factor of the planner's search is therefore the total number of
registered actions, for every expanded state. This is the dominant cost of planning and the
reason the visited-state budget is small. A rebuild that indexes operators by the properties
they affect turns this into a much smaller candidate set and is a legitimate improvement — but
it changes which of several equal-cost plans is found, and the game's behaviour is tuned
against the order operators were registered in.

### `value`

**Contract** — given a state and an operator, produces the successor state and records whether
the operator was applicable at all. Forward: tests the operator's preconditions against the
state and the measured world, and if they hold, applies its effects. Backward: tests that the
operator does not contradict the goal state, and if so, regresses the goal through it —
additionally rejecting a regression that changes nothing. The successor is written into a
single reused scratch state, so the caller must consume it before asking for the next edge.

### `is_accessible`

**Contract** — answers whether the immediately preceding `value` produced a real successor.

**Notes** — this is a *stateful pair*: `value` and `is_accessible` must be called in that order
for the same edge, and the engine does exactly that. It is the least defensible interface in
the chapter; a rebuild should have the successor routine return an optional state and delete
this entirely.

### `get_edge_weight`

**Contract** — the operator's cost between two states, and the place the operator-cost contract
is enforced: a cost below the operator's own lower bound is a hard failure, not a clamp.

**Notes** — the two states are passed to the operator in the *opposite* order to the one they
arrive in. That is deliberate for the backward search, where the edge runs from the later state
to the earlier one; the default cost ignores both states, so it is invisible until a concrete
action overrides `weight` and reads them.

### `is_goal_reached`

**Contract** — direction-dependent, and the two are not mirror images.

*Forward*: the goal is reached when every property of the target state holds — resolved against
the vertex delta first, then against the measured world, measuring lazily. Properties the
target does not name are ignored.

*Backward*: the vertex is itself a regressed goal, and the search is done when every property
of that vertex is satisfied by the measured world — that is, when no requirement is left
outstanding.

### `estimate_edge_weight`

**Contract** — the heuristic, direction-dependent.

*Forward*: the number of target properties the vertex delta does not already state with the
right value. Properties absent from the delta all count.

*Backward*: the number of the vertex's requirements that the measured world does not meet,
measuring lazily.

**Invariants** — the backward estimate is admissible under the operator-cost contract: each
remaining requirement costs at least one unit to discharge, and an operator's cost is at least
the number of properties it changes.

**Notes** — the forward estimate is **not** admissible, and this is the single most consequential
imprecision in the planning layer. Because the vertex records only differences from the world,
a target property that the world *already satisfies* is absent from the delta and is counted as
outstanding. The estimate therefore overshoots by the number of goal properties that are
already true, and the forward search — which is the direction the game actually uses — can
return a plan that is not the cheapest. The search still terminates and still returns a valid
plan; it is optimality that is given up. A rebuild that wants optimal forward plans must resolve
each target property against the measured world here as `is_goal_reached` already does, and
should expect more states to be expanded when it does.

## `add_operator` / `remove_operator` / `get_operator`

**Contract** — registration by identifier into the ordered operator list; duplicate registration
is a hard failure. In debug builds, registration also checks that every property the operator's
preconditions and effects name has an evaluator, and names the offending property when it does
not. Removal destroys the operator — the planner owns them — and swallows a failure during
destruction rather than letting it escape. Any registration or removal clears the staleness
flag.

**Notes** — swallowing a destruction failure is a C++ artifact of a codebase that disables
exceptions unevenly; the decision it encodes is "teardown must complete even if one operator
misbehaves", which a rebuild expresses directly.

## `add_evaluator` / `remove_evaluator` / `evaluator` / `evaluators`

**Contract** — registration of a property evaluator by property identifier. Duplicate
registration and lookup of an unregistered property are both hard failures: a property with no
way to measure it is a modelling error, not a runtime condition. Removal destroys the evaluator
and clears the staleness flag.

**Notes** — registering an evaluator does *not* clear the flag, only removing one does. That
asymmetry looks like an oversight: adding an evaluator for a property no operator mentions
changes nothing, which may be the reasoning, but adding one for a property that was missing
certainly does.

## `set_target_state` / `current_state` / `target_state`

**Contract** — setting the goal clears the staleness flag when the new goal differs from the
old, and leaves it alone when it does not — so re-asserting the same goal every frame, which is
what the brain loop does, costs nothing.

## `setup` / `init` / `clear`

**Contract** — `setup` returns the planner to the empty state: no goal, no measured world, no
plan, not stale, not failed. `init` is an empty hook for derived planners. `clear` releases
every operator and every evaluator, in reverse registration order.
