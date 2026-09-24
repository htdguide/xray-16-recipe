# src/xrGame/property_evaluator_const.h

> An evaluator whose answer is fixed at construction: the way a planner is told a condition it cannot measure.

**Needs** — [`property_evaluator.h`](property_evaluator.h.md)
**Used by** — [`agent_manager_properties.h`](agent_manager_properties.h.md) · [`object_property_evaluators.h`](object_property_evaluators.h.md) · [`property_evaluator_script.cpp`](property_evaluator_script.cpp.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`UIMapWndActions.h`](ui/UIMapWndActions.h.md)
**Tier floor** — T3: returns a stored value

## Purpose

Not every question a plan is written against is one the world can answer. Some are facts
about the *configuration* of a particular creature: *does this creature have a weapon at
all*, *is this a scripted actor*. A constant evaluator answers those, and lets the planner
treat them identically to measured questions — no special case in the search.

It also serves as the way a script installs a permanently-true or permanently-false
condition to disable a branch of a plan.

## State

```text
RECORD ConstEvaluator          # extends Evaluator
  value : bool                 # the answer, fixed at construction
```

## `evaluate`

**Contract** — returns the stored value. Never reads the subject or the storage, so a
constant evaluator is legal to evaluate before `setup` has ever run.

**Notes** — the constructor takes the answer and an optional name; unlike the base it does
not take a subject, because there is nothing to ask. This is the one evaluator whose
storage may legitimately stay unbound for its whole life.
