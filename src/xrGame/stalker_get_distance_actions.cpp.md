# src/xrGame/stalker_get_distance_actions.cpp

> Closing the range in bounds: run to the next piece of cover, breathe, run again.

**Needs** — [`stalker_get_distance_actions.h`](stalker_get_distance_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`cover_evaluators.h`](cover_evaluators.h.md) · [`cover_point.h`](cover_point.h.md) · [`stalker_movement_restriction.h`](stalker_movement_restriction.h.md)
**Used by** — [`stalker_get_distance_actions.h`](stalker_get_distance_actions.h.md)
**Tier floor** — T2: one cover-database search per bound.

## Purpose

Despite the name, this pair is about *closing* distance, not opening it: the branch runs
when the enemy is too far away for the creature's weapon to be useful. The behaviour is
classic bounding — a run to the next piece of cover that is nearer the enemy, a short pause,
and a re-evaluation. The pause is what makes it look like a tactical advance rather than a
charge, and it is one to three seconds long.

## State

`Stateless.`

## `RunToCover` — entry

**Contract** — pick a cover point between here and the enemy, and set off at a run. The
point is chosen once, on entry, not re-chosen per cycle: a bound is committed to.

```text
FUNCTION initialize()
  base.initialize()
  movement.mental_state := danger
  movement.gait         := run
  movement.body_state   := standing
  weapon_goal(IDLE, best_weapon)
  sight := watch along the path

  remembered := my memory of the selected enemy
  target     := remembered.position

  # "close" cover: protective, and nearer the enemy than I am now
  configure the close-cover evaluator with target,
      minimum standoff 0, maximum standoff = my current distance to target, spread 10
  point := best_cover(near = my position, radius = 10, close evaluator)
  IF point is none THEN retry with radius 30

  IF point exists THEN
    steer to point
  ELSE IF the enemy's own navigation vertex is somewhere I may go THEN
    steer to the enemy's remembered position           # no cover: go at him
  ELSE
    head for the nearest position to the enemy that I may occupy
```

**Invariants** — the maximum standoff handed to the evaluator is the creature's *current*
distance to the enemy. That single parameter is what makes every bound monotonic: a point
is only acceptable if it is closer than where the creature stands, so the branch cannot
oscillate and cannot retreat. A rebuild that passes a fixed radius here will produce
creatures that shuffle sideways forever.

**Notes** — the two fallbacks are ordered by desperation: cover if any exists, otherwise the
enemy's own position if it is legally reachable, otherwise the nearest legal approximation
of it. The middle case is what makes a stalker with no cover available advance into the
open rather than stand still, which is the correct behaviour when its weapon is useless at
this range anyway.

The weapon is idled rather than aimed at entry, because the creature is about to sprint;
aiming is picked up per cycle only if the enemy becomes visible.

## `RunToCover` — per cycle

**Contract** — fire on the move when the enemy is actually visible, otherwise keep the eyes
on the path; report arrival.

```text
FUNCTION execute()
  base.execute()
  IF the enemy is not visible right now THEN
    weapon_goal(IDLE, best_weapon)
    sight := watch along the path
  ELSE
    sight := watch the enemy
    fire()                       # the aim gate in the combat action base applies
  IF path not completed THEN RETURN
  set property InCover = true
```

**Invariants** — the visibility test is *visible now*, not remembered. A creature bounding
toward a remembered position does not shoot at the memory.

**Notes** — firing goes through the shared gate in
[`stalker_combat_action_base.cpp`](stalker_combat_action_base.cpp.md), so a creature whose
head has not caught up with the enemy keeps turning instead of shooting sideways while it
runs. The visual effect is a stalker that sprints, snaps a burst off when the enemy comes
into view, and keeps running.

## `WaitInCover` — entry

**Contract** — stop, pick a posture, idle the weapon, look along the path, and set a short
random timer.

```text
FUNCTION initialize()
  base.initialize()
  movement.mental_state := danger
  movement.gait         := stand still
  movement.body_state   := IF currently standing THEN randomly crouch or stand
                           ELSE crouch
  weapon_goal(IDLE, best_weapon)
  sight := clear, then watch along the path without turning the body
  inertia_time := uniform random between 1 and 3 seconds
```

**Invariants** — the posture rule is asymmetric on purpose: a creature already crouched
stays crouched, and only a standing creature flips a coin. Crouching is sticky because
standing up in cover is what gets a stalker shot, and because repeatedly toggling posture
reads as indecision.

**Notes** — the sight is *cleared* before being set, which is the only place in the danger
and distance branches that does so. Clearing discards whatever aiming intent the run
accumulated, so the pause starts from a neutral head position rather than continuing to
track an enemy the creature can no longer see.

One to three seconds is short. The pause exists to break the run into bounds, not to be a
hold; the hold-position behaviour belongs to a different branch and waits five to ten.

## `WaitInCover` — per cycle and exit

**Contract** — end the pause when the timer expires *or* the enemy becomes visible,
whichever is first. On exit, clear the cover proposition again and invalidate the creature's
remembered best cover so the next bound searches afresh.

```text
FUNCTION execute()
  base.execute()
  IF timer not elapsed AND the enemy is not visible now THEN RETURN
  set property InCover = false

FUNCTION finalize()
  base.finalize()
  set property InCover = false
  invalidate my cached best cover
```

**Invariants** — clearing `InCover` in both places is not redundancy. The per-cycle clear is
the normal transition to the next bound; the exit clear covers being interrupted by a plan
switch, after which the creature must not resume believing it is in cover it has left.

**Notes** — the enemy becoming visible cuts the pause short, which is what keeps the advance
responsive: the moment there is something to shoot at, the branch moves on rather than
spending its remaining two seconds crouched behind a crate.

Invalidating the cached cover at exit is what forces the *next* `RunToCover` to run a fresh
search with a new maximum standoff. Without it the creature would bound once and then keep
re-selecting the point it is already standing on.
