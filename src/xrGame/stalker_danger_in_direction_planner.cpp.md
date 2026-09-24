# src/xrGame/stalker_danger_in_direction_planner.cpp

> A five-step chain of pure progress propositions: nothing here is computed from the world except whether the danger still exists.

**Needs** — [`stalker_danger_in_direction_planner.h`](stalker_danger_in_direction_planner.h.md) · [`stalker_danger_in_direction_actions.h`](stalker_danger_in_direction_actions.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_danger_in_direction_planner.h`](stalker_danger_in_direction_planner.h.md)
**Tier floor** — T2: a five-operator search per cycle.

## Purpose

The tactical sequence a stalker runs against a threat it can face. Structurally it is the
simplest planner in the brain: a linear chain where each step's effect is the next step's
precondition, and every proposition but one is latched by an action rather than derived.

The contrast with the unknown-danger and grenade branches is worth stating. Those branches
keep an expensive `CoverActual` evaluator that re-searches the cover database every cycle
to decide whether the creature's chosen point is still right. This one does not: each of
its actions runs its own cover search inside its own execution, with its own evaluator, and
the planner never asks about cover at all. The consequence is that this branch cannot
*notice* that its cover has become invalid — it can only be interrupted from above, when the
danger expires or an enemy appears. That is a real behavioural difference and a rebuild
should reproduce it rather than "fix" it: a creature that re-decided its cover mid-flank
would never complete a flank.

## State

`Stateless.`

## `setup`

**Contract** — bind and rebuild both tables. Idempotent.

## `initialize`

**Contract** — on entering the branch: release the cover claim with the squad coordinator,
and clear all four progress propositions so the chain restarts at the top.

**Invariants** — clearing all four rather than some is what makes re-entering the branch
mean "react from the beginning". A creature that took cover against one shot and is then
shot at from elsewhere starts over rather than resuming at the flank.

## `add_evaluators`

```text
Danger          : is a danger still selected   (computed — the only real question here)
InCover         : am I behind something        (latched)
LookedOut       : have I got eyes on           (latched)
PositionHolded  : have I waited                (latched)
EnemyDetoured   : have I flanked               (latched)
```

**Notes** — four of the five read back what an action wrote. A planner over latched
propositions is a state machine wearing a planner's clothes, and here that is the right
shape: the sequence is genuinely fixed, and what the planner buys is not choice but
*uniform interruption* — the same machinery that abandons a plan when an enemy appears
works here without this branch knowing about it.

## `add_actions`

```text
TakeCover     requires  InCover = false
              effects   InCover = true

LookOut       requires  InCover = true,        LookedOut = false
              effects   LookedOut = true

HoldPosition  requires  LookedOut = true,      PositionHolded = false
              effects   PositionHolded = true

Detour        requires  PositionHolded = true, EnemyDetoured = false
              effects   EnemyDetoured = true

Search        requires  EnemyDetoured = true
              effects   Danger = false
```

**Invariants** — each step's negative precondition on its own effect is what stops the
planner choosing an action that is already satisfied, and is why the chain advances rather
than cycling. `TakeCover` is the only one whose precondition can be restored by another
action: hold-position clears `InCover` when it completes, which is what hands control to
the flanking step rather than back to cover.

**Notes** — the branch ends at `Search`, which resolves the danger by forgetting the object
that caused it. See
[`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) —
the distinction between forgetting one object and retiring the whole danger history is the
part a rebuild most easily gets wrong.

## `update` and `finalize`

**Contract** — pure delegation.
