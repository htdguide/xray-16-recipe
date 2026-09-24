# src/xrGame/action_planner_action_inline.h

> Makes a sub-plan behave as a single action: its goal is its own declared effect, its update is one step of its inner plan, and its inertia is the action's.

**Needs** — [`action_planner_action.h`](action_planner_action.h.md) · [`action_planner.h`](action_planner.h.md) · [`action_base.h`](action_base.h.md)
**Used by** — [`action_planner_action.h`](action_planner_action.h.md)
**Tier floor** — T2: one inner decision cycle per outer execute

## Purpose

Six short methods, each of which answers one question about how a nested brain behaves
when the brain above it is driving. Together they are the whole of hierarchical planning
in this engine, and every one of them is a decision a rebuild must also make.

## State

Adds nothing. The composite's state is its planner half's (world-state storage, running
sub-action) plus its action half's (inertia window, cost, first-update flag).

Invariant: the two halves are bound to the *same* acting object and the outer planner's
storage is *not* the inner one's — see setup.

## `setup`

**Contract** — binds both halves to the acting object, then sets the inner planner's goal
to this action's own declared effects.

**Invariants** — this is the load-bearing line of the file: **a sub-plan's goal is exactly
what the sub-plan promises the outer plan it will achieve.** The author declares the
composite's effects once, as an action, and the inner search is automatically aimed at
them. Nothing can drift, because there is only one declaration.

```text
FUNCTION setup(object, outer_storage)
  planner_half.setup(object)              # fresh inner storage, no running sub-action
  action_half.setup(object, outer_storage)
  planner_half.goal_state = action_half.effects
```

**Notes** — the planner half's setup clears its *own* world-state storage, which the inner
actions and evaluators share. The outer storage passed in binds only the action half. So
an inner evaluator cannot see an outer property: the two levels have separate world states
and communicate only through the effects declaration. That isolation is deliberate and a
rebuild that shares one storage across levels will get different behaviour.

## `execute`

**Contract** — runs the action half's execute (which lowers the first-update flag), then
one full decision cycle of the inner planner: re-solve, switch the inner action if its
head changed, execute it.

**Invariants** — the inner plan is recomputed on *every* outer execute, at exactly the
same cadence as the outer plan. Nesting therefore costs one extra graph search per level
per update; the hierarchies in the shipped game are two or three deep, which is the
practical bound.

## `initialize`

**Contract** — runs the action half's initialize only. The inner planner deliberately does
*not* reset: it re-solves on the first execute anyway, and resetting would throw away the
running sub-action for no gain.

## `finalize`

**Contract** — the action half first, then the planner half — which finalizes whatever
sub-action was running and marks the inner brain uninitialized.

**Invariants** — the order matters and is the mirror of construction. Finalizing the inner
brain first would run a sub-action's closing code while the outer action still considered
itself running.

## `completed`

**Contract** — the action half's answer alone: has this action's inertia window elapsed.
Explicitly *not* "has the inner plan reached its goal". A sub-plan's completeness is
judged from outside, by the outer planner's own evaluators reading the world, not by the
sub-plan reporting on itself.

## `add_condition` / `add_effect`

**Contract** — forwarded to the planner half, so declaring a *sub*-action's precondition
goes through the inner planner's "not while solving" guard.

## `save` / `load`

**Contract** — planner half first (its evaluators, its actions, its storage), then action
half. Positional, like every part of the brain's persistence.
