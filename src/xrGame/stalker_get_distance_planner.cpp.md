# src/xrGame/stalker_get_distance_planner.cpp

> Two operators that alternate forever, until the enemy is close enough to shoot.

**Needs** — [`stalker_get_distance_planner.h`](stalker_get_distance_planner.h.md) · [`stalker_get_distance_actions.h`](stalker_get_distance_actions.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_get_distance_planner.h`](stalker_get_distance_planner.h.md)
**Tier floor** — T2: a two-operator search per cycle.

## Purpose

The bounding advance, expressed as a two-state loop. It is the smallest planner in the
brain that *cycles* rather than terminating, and the cycle is the interesting part: neither
action ends the branch, and the branch is left only when the world changes underneath it.

## State

`Stateless.`

## `setup`

**Contract** — bind, clear the cover proposition, rebuild both tables, and **invalidate the
creature's cached best cover**.

```text
FUNCTION setup(creature, parent_storage)
  base.setup(creature, parent_storage)
  inner.InCover := false
  clear evaluators and operators
  add_evaluators()
  add_actions()
  creature.invalidate_cached_best_cover()
```

**Invariants** — the cache invalidation is what guarantees the first bound searches from
scratch. Without it a creature entering this branch would inherit a cover point chosen for
a completely different purpose by whatever ran before.

## `add_evaluators`

```text
InCover           : latched by the two actions
TooFarToKillEnemy : computed — is the enemy beyond my weapon's useful range
```

**Notes** — the useful range is the creature's own, derived from its weapon, so the branch
means different distances for a shotgun and a rifle. That is what makes a shotgun-armed
stalker rush and a rifleman hold.

## `add_actions`

```text
RunToCover   requires  InCover = false
             effects   InCover = true

WaitInCover  requires  InCover = true,  TooFarToKillEnemy = true
             effects   TooFarToKillEnemy = false
```

**Invariants** — the pair is a two-state cycle: running sets `InCover`, waiting clears it
again from inside its own execution, and the plan re-enters at the run. Neither action
writes `TooFarToKillEnemy`, which is computed from the actual distance, so the branch
terminates when and only when the creature has genuinely closed the range. The effect
`WaitInCover` claims is again the planner idiom for "running this makes the world satisfy
that", not an assignment.

**Notes** — the goal the parent asks for is `TooFarToKillEnemy = false`, and the only
operator producing it is the wait. So the plan the search finds is always *run, then wait*,
and the loop comes from the wait resetting its own precondition rather than from any loop
construct. Understanding that is the difference between reading this planner as two lines
of configuration and reading it as a bounding advance.
