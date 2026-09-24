# src/xrAICore/Components/operator_abstract_inline.h

> One action in the planner's vocabulary — what it requires, what it changes, what it costs — and the four state-algebra routines that let the search step forward from the world or backward from the goal.

**Needs** — [`operator_abstract.h`](operator_abstract.h.md) · [`condition_state_inline.h`](condition_state_inline.h.md) · [`problem_solver_inline.h`](problem_solver_inline.h.md)
**Used by** — [`operator_abstract.h`](operator_abstract.h.md) · [`problem_solver_inline.h`](problem_solver_inline.h.md)
**Tier floor** — T2: merge walks over ordered property lists.

## Purpose

This file holds the operator semantics of the goal/plan/action layer. Everything else in the
planner is bookkeeping around what these four routines decide: whether an action is legal at
a point in the search, and what the world looks like on the far side of it.

Two things here are easy to get wrong and expensive to get wrong.

The first is the **delta representation**. A vertex of the planner's search is not a full
world; it records only the properties whose value *differs from the measured world*. The start
vertex is therefore the empty state, and applying an operator can make a state smaller.
Everything in `applicable` and `apply` below is written around that.

The second is the **lazy evaluation of the world**. The planner never measures the world up
front. When a routine here needs a property that neither the vertex delta nor the already
measured properties mention, it calls back into the solver, which runs that property's
registered evaluator and caches the answer. Only the properties a search actually consults are
ever measured — which is why a property evaluator is allowed to be expensive.

## State

```text
RECORD Operator
  conditions    : WorldState      # preconditions: partial, what the world must look like
  effects       : WorldState      # what the world looks like afterwards; also partial
  actuality     : optional<ref to bool>   # the owner's "current plan is still valid" flag
  min_weight    : int             # cached lower bound on cost
  weight_actual : bool            # is min_weight current
```

**Invariants**

- `min_weight` is a *lower bound* on whatever `weight` returns. The planner asserts this on
  every edge it relaxes, and it is not a sanity check — see below.
- Editing either state clears `weight_actual` and clears the owner's actuality flag. An
  operator whose definition changed cannot leave a plan built from the old definition standing.
- Both states obey the world-state invariant: ascending by property identifier, one entry per
  property.

## `min_weight`

**Contract** — the number of properties this operator actually changes: an effect on a property
the preconditions do not mention counts one, an effect that disagrees with the precondition on
the same property counts one, an effect that agrees counts nothing. Computed by one merge walk
the first time it is asked for after an edit, then cached. Reading it is free thereafter.

```text
FUNCTION min_weight() -> int
  IF weight_actual THEN RETURN cached
  n <- 0
  walk conditions and effects together in ascending property order
    property in conditions only                   -> ignore
    property in effects only                      -> n <- n + 1
    property in both, values differ               -> n <- n + 1
    property in both, values agree                -> ignore
  n <- n + (effects not yet reached)              # effects outlast conditions
  cache n; weight_actual <- true; RETURN n
```

**Notes** — this is not an optimization, it is what makes the planner's heuristic admissible.
The heuristic estimates the remaining cost as *the number of goal properties not yet
satisfied*. For that estimate never to exceed the truth, one unit of cost must buy at most one
satisfied property — which is exactly "cost is at least the number of properties changed". The
planner asserts `weight >= min_weight` on every edge, and a concrete action that returns a
cheaper cost silently makes the search return non-optimal plans.

## `weight`

**Contract** — the cost of taking this operator between two states. The default ignores both
states and returns `min_weight()`. Concrete actions override it to express "this action is
expensive right now" — but may only ever return a value at or above `min_weight()`.

## `setup` / actuality

**Contract** — binds the flag that the operator clears whenever its definition changes, and
clears it once at bind time. The flag belongs to the planner instance, which reads it to decide
whether the standing plan can be kept. An operator with no flag bound simply does not report.

**Notes** — the clearing routine is written as an AND-accumulate and is only ever called with
"false", so it is in practice "invalidate". A rebuild should just call it `invalidate`.

## `applicable` — forward search

**Contract** — can this operator's preconditions be met at a search vertex? The vertex carries
only the properties that differ from the world, so each precondition is resolved in three
steps: consult the vertex delta; failing that, consult the measured world; failing that, ask
the solver to measure it now and cache it. One merge walk in which all three sequences advance
monotonically, so each property is resolved once.

```text
FUNCTION applicable(vertex_delta, measured_world, preconditions, solver) -> bool
  FOR EACH required IN preconditions, in ascending property order
    IF vertex_delta names required.property
      value <- vertex_delta value                  # the search changed it
    ELSE
      advance measured_world to required.property
      IF measured_world does not name it
        solver.evaluate(required.property)         # measure it now, cache it in measured_world
        value <- the freshly measured value
      ELSE
        value <- measured_world value
    IF value != required.value THEN RETURN false
  RETURN true
```

**Notes** — the routine is written as two loops because the vertex delta and the measured world
are two separate ordered sequences consulted in priority order; once the delta is exhausted,
the remaining preconditions are resolved against the world alone. Merging them is the obvious
rebuild and changes nothing observable.

Measuring a property *inserts* it into the solver's measured-world list, which can move the
list in memory — the routine re-derives its cursors after every such call. That is the
incidental C++ made explicit: a rebuild whose sequence type does not invalidate positions can
drop it, but must still keep the cursors in step with an insertion before them.

## `apply` — forward search

**Contract** — the successor vertex. Produces the delta of the post-state against the measured
world, given the pre-state delta and the operator's effects. A property survives into the
result only when it still differs from the world.

```text
FUNCTION apply(vertex_delta, effects, result, measured_world, solver) -> WorldState
  result <- empty
  FOR EACH property mentioned by vertex_delta or effects, ascending
    only in vertex_delta                      -> keep the delta entry
    only in effects                           -> resolve the property against measured_world
                                                 (measuring it if needed);
                                                 keep the effect ONLY IF it differs from it
    in both, values agree                     -> keep it                 # still differs from world
    in both, values disagree                  -> drop it                 # the effect undid the delta
  RETURN result
```

**Invariants** — the result is built by appending in ascending order, which the merge walk
guarantees. The empty result is the meaningful case: it means this vertex is the world as it
actually is.

**Notes** — dropping a property when the effect disagrees with the delta is the non-obvious
line and the one that makes the delta representation work. The delta says "the search has made
this property differ from the world"; an effect that disagrees with the delta is setting it
back to what the world already says, so the difference disappears.

## `applicable_reverse` — backward search

**Contract** — in a backward search the vertex is a *goal*: a partial state that must hold. An
operator may be regressed through it when it does not contradict it — neither its effects nor
its preconditions may name a property the goal names with a different value. Properties the
goal names and the operator does not touch are fine; they simply survive the regression.

```text
FUNCTION applicable_reverse(effects, preconditions, goal) -> bool
  FOR EACH required IN goal, ascending
    IF effects name it with a different value       -> RETURN false
    IF effects do not name it
       AND preconditions name it with a different value -> RETURN false
  RETURN true
```

## `apply_reverse` — backward search

**Contract** — regresses a goal through this operator: the new goal is the old one with the
properties this operator's effects achieve removed, and the operator's preconditions added.
Reports whether the regression *changed anything*; a regression that does not is rejected by
the caller, because an operator that neither discharges a goal property nor adds a requirement
would generate a self-loop the search could follow forever.

```text
FUNCTION apply_reverse(goal, effects, result, preconditions) -> bool
  result  <- empty
  changed <- false
  FOR EACH property mentioned by goal or preconditions, ascending
    in goal only
      IF effects achieve it  -> drop it; changed <- true     # this operator supplies it
      ELSE                   -> keep it                      # still required upstream
    in preconditions only    -> add it                       # a new requirement
    in both                  -> take the precondition's value;
                               changed <- true when the two disagree
  RETURN changed
```

**Notes** — the "effects achieve it" test asserts that the effect's value matches the goal's;
that combination was already screened by `applicable_reverse`, so reaching it with a mismatch
is a contract violation rather than a case.

## `apply` (the two-state merge)

**Contract** — merges two partial states into a result: every property of either, with the
second argument winning where both name the same property. Used to fold an operator's
preconditions into a goal.

**Notes** — the "second wins" tie-break is asserted rather than chosen, because at every call
site the two agree by construction. A rebuild should still pick a side explicitly.
