# src/xrGame/stalker_low_cover_planner.cpp

> Pin the creature at the cover, then choose between ducking, shooting and watching — by whether it can see anything.

**Needs** — [`stalker_low_cover_planner.h`](stalker_low_cover_planner.h.md) · [`stalker_low_cover_actions.h`](stalker_low_cover_actions.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`stalker_planner.h`](stalker_planner.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_low_cover_planner.h`](stalker_low_cover_planner.h.md)
**Tier floor** — T2: a three-operator search per cycle.

## Purpose

The branch a stalker runs when the cover it is at protects it only while crouched. Its three
actions do not move the creature at all, which is unusual: the branch's contribution is
entirely posture and aim, and the *movement* decision was already made by whoever sent the
creature here.

## State

`Stateless.`

## `setup`

**Contract** — bind and rebuild both tables. Idempotent.

## `initialize`

**Contract** — on entering the branch, pin the creature: stand it at the nearest position it
may legally occupy, stop it moving, configure the path settings it would use if something
did move it, put it in an alert mental state, and declare it in cover.

```text
FUNCTION initialize()
  base.initialize()
  movement.gait := stand still
  movement.head_for_nearest_accessible_position()
  movement.desired_direction := none
  movement.path_type         := level path, smooth detail
  movement.mental_state      := danger
  inner.InCover := true
```

**Invariants** — pinning happens here, once, rather than in the actions. That is what frees
all three actions to be pure posture decisions, and it is why none of them touches movement.

## `update`

**Contract** — before each planning cycle, refresh the creature's cover hint with the
enemy's remembered position, then run the cycle.

```text
FUNCTION update()
  remembered := my memory of the selected enemy
  IF I remember it THEN best_cover_hint := remembered.position
  base.update()
```

**Notes** — the hint is also refreshed inside each action, so this is belt and braces. It
matters for the gap: the planner's update runs even on cycles where the plan is being
rebuilt and no action executed, and the hint must not go stale in that gap.

## `add_evaluators`

```text
LowCover    : constant true        # inside this branch, by definition
ReadyToKill : computed — is the weapon drawn, loaded and raised
SeeEnemy    : computed — can I see the enemy right now
```

**Notes** — `LowCover` is pinned true here for the same reason `PuzzleSolved` is pinned
false in the death branch: the branch's own precondition is a fact inside it, and the
constant is what lets the operators claim its negation as their terminating effect.

## `add_actions`

```text
GetReadyToKill  requires  ReadyToKill = false
                effects   ReadyToKill = true

KillEnemy       requires  ReadyToKill = true,  SeeEnemy = true
                effects   LowCover = false

HoldPosition    requires  ReadyToKill = true,  SeeEnemy = false
                effects   LowCover = false
```

**Invariants** — the two fighting actions are separated by exactly one proposition, and it
is the one the creature cannot control: whether it can see the enemy. So the posture choice
is driven by visibility rather than by a timer or a state machine — the creature stands and
shoots while it has a target, and stands and watches while it does not. A rebuild that adds
hysteresis here will smooth out a behaviour whose jitteriness is doing useful work: a
stalker at low cover bobbing between shooting and watching is what reads as a firefight.

Both fighting actions claim `LowCover = false` as their effect, and neither writes it. The
branch ends when the parent's own evaluator stops reporting low cover — because the creature
moved, the enemy died, or the situation changed — not because anything here decided it had.
See [`stalker_low_cover_actions.cpp`](stalker_low_cover_actions.cpp.md) for the one place
the hold action does reach upward, which is a separate and less defensible mechanism.

## `execute` and `finalize`

**Contract** — pure delegation.
