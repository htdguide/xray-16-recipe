# src/xrGame/stalker_kill_wounded_planner.cpp

> Five steps, one of which may only start once, and a flag telling the combat planner not to interfere.

**Needs** — [`stalker_kill_wounded_planner.h`](stalker_kill_wounded_planner.h.md) · [`stalker_kill_wounded_actions.h`](stalker_kill_wounded_actions.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_kill_wounded_planner.h`](stalker_kill_wounded_planner.h.md)
**Tier floor** — T2: a five-operator search per cycle.

## Purpose

The execution sequence, and the contract it has with the combat planner that contains it.
The interesting part is not the chain — that is the same shape as every other danger and
combat sub-planner — but the way the branch declares itself *in progress* to its parent, so
that a creature standing over a wounded man is not pulled back into general combat halfway
through saying its line.

## State

`Stateless.` Two storages, as with every sub-planner: its own, and the parent's.

## `setup` and `initialize`

**Contract** — `setup` binds, clears the prepared proposition, and rebuilds both tables.
`initialize` clears all three latched propositions and then **writes `KilledWounded = true`
into the parent's storage**.

```text
FUNCTION initialize()
  base.initialize()
  inner.WoundedEnemyPrepared := false
  inner.WoundedEnemyAimed    := false
  inner.PausedAfterKill      := false
  outer.KilledWounded        := true      # tell the combat planner I am busy
```

**Invariants** — the parent's flag is raised on *entry*, before any step has run, and not
by any action. It is a statement of intent, not of progress: the combat planner reads it to
decide whether this creature is currently engaged in an execution, and the whole point is
that the answer is yes from the first frame.

## `finalize`

**Contract** — on leaving the branch, if there is still an enemy selected, lower the
parent's flag and put the creature back into an alert mental state.

```text
FUNCTION finalize()
  base.finalize()
  IF an enemy is still selected THEN
    outer.KilledWounded   := false
    movement.mental_state := danger
```

**Invariants** — the guard matters. Leaving the branch with no enemy means the execution
succeeded and the creature is done; leaving it with an enemy still selected means it was
interrupted — another threat appeared — and the creature must be handed back to combat
alert rather than left in the relaxed state the approach action set.

**Notes** — restoring the mental state here rather than in the next action is the only way
to cover both exits. The approach action deliberately sets the creature at ease, and an
interruption that skipped this line would leave a stalker strolling into a firefight.

## `add_evaluators`

```text
Enemy                 : is there an enemy, or was there recently   (computed, with the
                                                                    same post-combat delay
                                                                    the combat planner uses)
WoundedEnemyReached   : am I close enough                          (computed)
WoundedEnemyPrepared  : has the line been delivered                (latched)
WoundedEnemyAimed     : is the head on target                      (latched)
PausedAfterKill       : is the post-kill beat still owed           (latched)
```

## `add_actions`

```text
ReachWounded    requires  PausedAfterKill = false, Enemy = true,
                          WoundedEnemyReached = false
                effects   WoundedEnemyReached = true

AimWounded      requires  PausedAfterKill = false, WoundedEnemyReached = true,
                          WoundedEnemyAimed = false
                effects   WoundedEnemyAimed = true          settle time 1 s

PrepareWounded  requires  PausedAfterKill = false, WoundedEnemyReached = true,
                          WoundedEnemyAimed = true, WoundedEnemyPrepared = false
                effects   WoundedEnemyPrepared = true

KillWounded     requires  WoundedEnemyReached = true, WoundedEnemyPrepared = true,
                          WoundedEnemyAimed = true
                effects   Enemy = false

PauseAfterKill  requires  PausedAfterKill = true
                effects   PausedAfterKill = false            pause 1 s
```

**Invariants** — the first three steps all carry `PausedAfterKill = false` as a
precondition, and the kill step does not. That asymmetry is the mechanism of the post-kill
beat: the kill action raises the pause flag on entry, which immediately disqualifies the
three approach steps, leaving `PauseAfterKill` as the only available operator once the kill
completes. There is no sequencing anywhere — the pause is forced by making everything else
unplannable.

`KillWounded` also omits the pause precondition so that it can run on the same cycle it
raises the flag.

**Notes** — the two timed steps have their intervals configured *here*, by the planner, not
inside the actions. Both are one second: the aim settle and the post-kill beat. Keeping them
on the planner means the rhythm of the scene is visible in one place, which is the right
call for a sequence whose whole value is its pacing.

## `update` and `execute`

**Contract** — pure delegation.

**Notes** — an abandoned refinement survives here as commented-out code: it would have
raised the parent's `KilledWounded` flag only while the kill action itself was running,
rather than for the whole branch. The shipped behaviour — the flag up for the entire
branch — is the one that protects the approach and the voice line, which is what the flag
exists for. There is nothing to recover.
