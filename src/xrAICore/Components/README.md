# src/xrAICore/Components — the goal, plan and action layer

> A world state is a set of property/value pairs, an action declares what it requires and
> what it changes, and planning is a search over actions from the world as measured to a goal
> state. The search itself is borrowed from the navigation chapter.

Part of [chapter 14](../README.md).

## What this directory is responsible for

Four types and their script surface.

A **world property** is one question about the world bound to one answer: an identifier and a
value. Which questions exist is not decided here — chapter 24 names them.

A **world state** is a set of those pairs, kept sorted so that two states can be compared,
merged and subtracted by a single merge walk, and carrying a hash that is the XOR-fold of its
properties' hashes.

An **operator** — an action — declares a precondition state, an effect state and a cost.
Applying it to a state, testing whether a state satisfies its preconditions, and the backward
equivalents of both, are four state-algebra routines and are the only interesting code in the
type.

The **planner** holds the operator set, the set of evaluators that measure properties, the
current goal, and the standing plan. It decides when the plan has gone stale, and it presents
itself to the path-search engine as a graph whose vertices are world states and whose edges
are operators — which is why there is no search algorithm in this directory.

## Where it sits

It rests on [`Navigation/`](../Navigation/README.md) for the search engine and its adapter
policy, on the core layer for containers, and on the script engine for the two types mods
build goals out of. Its consumer is chapter 24: every concrete evaluator, every concrete
action and every creature brain is written there, in the vocabulary defined here.

## The load-bearing ideas

**A world state is a delta, not a snapshot.** A search vertex records only the properties
whose value differs from the world *as measured*. The start vertex of a forward search is
therefore the empty state; applying an action can make a state smaller; and two different
action sequences that reach the same world reach the same vertex, which is what makes the
visited-state table worth having. The hash is maintained incrementally and is
order-independent, so it survives insertion and removal without recomputation.

**The world is measured lazily and cached for the life of one plan.** No property is
evaluated until some branch of the search asks about it. The answer is then cached, and the
set of properties consulted becomes the plan's dependency set. Re-planning is triggered by
re-measuring exactly that set and seeing whether any answer moved. A world change that no
standing plan depended on costs nothing — which is what makes it affordable to run a planner
per creature per frame.

**An action's cost is bounded below by what it changes.** Every action reports a lower bound
equal to the number of properties it actually changes, and any cost it declares must be at or
above it. This is not a sanity check. The planner's heuristic counts unsatisfied goal
properties, so one unit of cost must buy at most one satisfied property, or the estimate stops
being a bound at all.

**The planner can search in either direction, and the shipped direction is forward.** Both
the forward and backward state-algebra routines exist on the operator. Forward search starts
from the empty delta and applies effects; backward search starts from the goal and regresses
preconditions. The brains in chapter 24 use the forward one.

**What is guaranteed and what is not.** *Guaranteed*: every action in the returned plan had
its preconditions satisfied at the point it appears, given the world as measured; the plan's
length is bounded by the visited-state budget; and a plan is never executed after a property
it relied on has changed. *Merely attempted*: that the plan is the cheapest one — the forward
heuristic counts every goal property absent from the delta as outstanding, including those the
world already satisfies, so it overestimates and the search will settle for a costlier plan.
*Not attempted at all*: that a plan exists. The search abandons after a bounded number of
visited states, and abandoning is indistinguishable from impossibility. A rebuild must
therefore give the brain a defined behaviour for "no plan", because it will happen for
reachable goals under load.

**Scripts build goals, not plans.** The property and the state are exported; the planner and
the operator are not. A mod names a target world state and the engine's operators find the way
there, which is what keeps the script surface small while leaving the interesting decision in
data.

## The twins

| Twin | Role |
|---|---|
| [`operator_condition.h`](operator_condition.h.md) | The atom: one property identifier bound to one value |
| [`operator_condition_inline.h`](operator_condition_inline.h.md) | The pair plus the derived hash that lets a whole state be hashed by XOR-folding |
| [`condition_state.h`](condition_state.h.md) | The world state: a sorted, hashed set of property/value pairs |
| [`condition_state_inline.h`](condition_state_inline.h.md) | The set algebra and the incremental, order-independent hash that makes state identity cheap enough to key a search vertex |
| [`operator_abstract.h`](operator_abstract.h.md) | An action: preconditions, effects, cost |
| [`operator_abstract_inline.h`](operator_abstract_inline.h.md) | The four merge-walk routines that step the search forward from the world or backward from the goal |
| [`problem_solver.h`](problem_solver.h.md) | The planner: an operator set, an evaluator set, a goal, and the graph face it shows the search engine |
| [`problem_solver_inline.h`](problem_solver_inline.h.md) | Lazy measurement, staleness detection, and the vertex/edge interface the search walks |
| [`script_world_property_script.cpp`](script_world_property_script.cpp.md) | Publishes the property to scripts as `world_property` |
| [`script_world_state_script.cpp`](script_world_state_script.cpp.md) | Publishes the state to scripts as `world_state`, so shipped scripts can build goals |

## What could not be recovered

- **The visited-state budget of 8000** is explicable — it sits just under a fixed pool of 8192
  — but the pool size is not. A comment in the search engine notes that the solver assembly is
  *constructed* for 16384 vertices while its manager and allocator are sized for 8192, and
  flags it as possibly a mistake. It is unresolved here.
- **`weight` on the world state** is declared taking a single property but its body needs a
  whole state; it compiles only because it is never instantiated. The planner computes the
  same quantity itself. Whether the method was meant to survive is unknown.
- **`property` on the world state** returns the next property at or after the requested one
  when the requested one is absent, rather than nothing. This is reachable from script.
  Whether callers were expected to check, or whether nobody noticed, is not recoverable.
