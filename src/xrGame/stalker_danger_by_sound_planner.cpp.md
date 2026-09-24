# src/xrGame/stalker_danger_by_sound_planner.cpp

> A placeholder branch: one evaluator hardwired to false, one operator the author labelled "fake".

**Needs** — [`stalker_danger_by_sound_planner.h`](stalker_danger_by_sound_planner.h.md) · [`stalker_danger_by_sound_actions.h`](stalker_danger_by_sound_actions.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_danger_by_sound_planner.h`](stalker_danger_by_sound_planner.h.md)
**Tier floor** — T2: a one-operator planner.

## Purpose

The fourth danger branch, wired into the danger planner and never reached. It is kept in
the recipe because a rebuilder comparing the danger planner's four operators against four
implementations must be told that this one is a stub, not that they missed a file.

## State

`Stateless.`

## `setup`

**Contract** — bind and rebuild, like every planner. Idempotent.

## `add_evaluators`

```text
Danger         : is a danger still selected      (real)
DangerUnknown  : constant false                  (the author's own label for it is "fake")
```

## `add_actions`

```text
TakeCover   requires  (nothing)
            effects   Danger = false
```

One operator, registered under the *unknown-danger* branch's operator identifier rather
than under one of its own, wrapping the take-cover action from
[`stalker_danger_by_sound_actions.cpp`](stalker_danger_by_sound_actions.cpp.md) — which
does nothing but stand still.

**Notes** — the branch is unreachable from above: the danger planner only selects it when
its `DangerBySound` classifier is true, and that classifier is hardwired false. Were it
reachable, this plan would put the creature into a wary standstill and immediately declare
the danger resolved.

A rebuild has three defensible choices: delete the branch and its five actions; implement
it and restore the enemy-sound danger type to its classifier, which is where the original
evidently intended that type to go; or reproduce the stub faithfully. Only the first two
are worth doing. The behaviour of the shipped game is unaffected either way.
