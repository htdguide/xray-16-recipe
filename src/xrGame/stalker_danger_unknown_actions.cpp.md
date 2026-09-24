# src/xrGame/stalker_danger_unknown_actions.cpp

> What a stalker does about a threat with no direction: take cover, crouch and sweep, then mark the spot so the squad avoids it.

**Needs** — [`stalker_danger_unknown_actions.h`](stalker_danger_unknown_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`object_handler.h`](object_handler.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_location_manager.h`](agent_location_manager.h.md) · [`danger_cover_location.h`](danger_cover_location.h.md) · [`cover_point.h`](cover_point.h.md)
**Used by** — [`stalker_danger_unknown_actions.h`](stalker_danger_unknown_actions.h.md)
**Tier floor** — T2: per-cycle movement and sight configuration plus one squad-wide publication.

## Purpose

A ricochet, a body dropping, a corpse found: something is wrong and there is no bearing to
face. The reaction is therefore *defensive and exploratory* rather than directed — get
behind cover, look everywhere, and then record that this place is dangerous so the squad
routes around it. The third step is the interesting one, because it is how a
single creature's fright becomes squad knowledge.

## State

Only the take-cover action holds anything: one boolean chosen at random on entry, deciding
what the creature watches while it runs.

## `DangerUnknownTakeCover` — entry

**Contract** — clear the two progress propositions, configure a cautious level-path
movement, and flip a coin for the sight mode.

```text
FUNCTION initialize()
  base.initialize()
  set property CoverReached  = false
  set property LookedAround  = false
  movement.desired_direction := none
  movement.path_type         := level path
  movement.detail_path_type  := smooth
  movement.mental_state      := danger
  direction_sight := random boolean
```

**Notes** — the coin flip is the file's one piece of pure characterization. Half the time
the creature runs looking where it is going; half the time it runs looking at the cover it
is heading for. Neither is better; having both is what stops a squad of six from moving
identically. This kind of per-invocation randomization appears throughout the creature layer
and is cheaper than any amount of behaviour authoring.

## `DangerUnknownTakeCover` — per cycle

**Contract** — steer toward the claimed cover point, hold the weapon aimed and ready, and
report arrival. Returns immediately if the danger has expired, leaving the creature's
current configuration alone so the planner can re-branch cleanly.

```text
FUNCTION execute()
  base.execute()
  IF no danger is selected THEN RETURN

  point := my claimed cover point
  IF point exists THEN
    movement.destination_vertex := point.level_vertex
    movement.desired_position   := point.position
  ELSE
    movement.head_for_nearest_accessible_position()

  weapon_goal(AIM_READY, best_weapon)

  IF path is not yet completed THEN
    movement.body_state := standing
    movement.gait       := run
    IF NOT direction_sight OR distance_to_destination <= 2 THEN
      sight := watch the cover point
    ELSE
      sight := watch along the path
    RETURN

  set property CoverReached = true
```

**Invariants** — the sight mode collapses to "watch the cover" in the last two units
regardless of the coin flip. A creature arriving at cover must be facing the cover, or the
crouch that follows puts its back to the thing it is hiding behind.

**Notes** — posture and gait are set inside the not-yet-arrived branch, so a creature that
has arrived keeps whatever the next action sets rather than being forced to stand. The run
is unconditional: unlike the anomaly escape, there is nothing about unknown danger that
rewards moving slowly.

When no cover point has been granted, the creature asks the movement layer for the nearest
place it is allowed to be. That is the degenerate case — no cover exists within reach — and
the behaviour is to at least stop standing somewhere illegal, then let the evaluator try
again next cycle.

## `DangerUnknownLookAround` — entry

**Contract** — crouch in place for a fixed interval and sweep. Sets the action's completion
delay to fifteen seconds *before* calling up, so the base action's timer starts from this
value rather than a default.

```text
FUNCTION initialize()
  inertia_time := 15 seconds       # must precede base.initialize
  base.initialize()
  movement.path_type        := level path
  movement.detail_path_type := smooth
  movement.gait             := stand still
  movement.mental_state     := danger
  movement.body_state       := crouch
```

**Invariants** — the ordering of the inertia assignment against the base call is
load-bearing and easy to get wrong; the base captures the current time against the current
inertia value.

## `DangerUnknownLookAround` — per cycle

**Contract** — sweep the eyes, and report the sweep finished once the interval elapses.

```text
FUNCTION execute()
  base.execute()
  IF no danger is selected THEN RETURN
  IF the body has finished turning to its target heading THEN
    sight := look over the top of the cover
  ELSE
    sight := watch the cover
  IF inertia has elapsed THEN set property LookedAround = true
```

**Notes** — the two sight modes are sequenced by the *body*, not by a timer: while the
creature is still rotating into position it keeps its eyes on the cover, and only once
settled does it raise its gaze over the top. That coupling is what makes the crouch read as
one continuous movement instead of a turn and an unrelated head animation.

## `DangerUnknownSearch`

**Contract** — the branch's terminator. If the creature holds a cover point, publish that
point to the squad as a *danger location* with a lifetime and a radius, and return with the
danger still standing. Otherwise, reset the two progress propositions so the branch starts
over.

```text
FUNCTION execute()
  base.execute()
  point := my claimed cover point
  IF point exists THEN
    squad.locations.add(DangerLocation(point, now, lifetime = 120 s, radius = 5))
    RETURN
  set property CoverReached = false
  set property LookedAround = false
```

**Invariants** — this action's declared effect is `Danger = false`, but it never writes it.
It ends the branch by *changing the world*: publishing the danger location is what makes
the creature's own danger evaluator stop selecting this threat. An action that declared an
effect and then asserted it directly would end the branch even when the publication failed.

**Notes** — the published record is the mechanism by which a scare becomes squad-wide
avoidance. Every member's pathing consults these locations, so one stalker hearing a
ricochet near a doorway keeps the whole squad out of that doorway for two minutes. The
radius is five units — roughly a room — and the lifetime two minutes, long enough to outlast
the engagement that caused it and short enough that the level does not fill up with stale
fear.

The no-cover branch resetting both propositions is the recovery path: without a cover point
there is nothing to publish, so the branch restarts at take-cover rather than terminating
on a threat nobody has actually reacted to.
