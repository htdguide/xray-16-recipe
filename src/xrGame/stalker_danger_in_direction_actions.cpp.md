# src/xrGame/stalker_danger_in_direction_actions.cpp

> Five reactions to a threat with a bearing, and the three different cover evaluators that give each of them its character.

**Needs** — [`stalker_danger_in_direction_actions.h`](stalker_danger_in_direction_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`object_handler.h`](object_handler.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`cover_evaluators.h`](cover_evaluators.h.md) · [`cover_point.h`](cover_point.h.md) · [`stalker_movement_restriction.h`](stalker_movement_restriction.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_location_manager.h`](agent_location_manager.h.md)
**Used by** — [`stalker_danger_in_direction_actions.h`](stalker_danger_in_direction_actions.h.md)
**Tier floor** — T2: several cover-database searches per creature per cycle.

## Purpose

This is the richest of the four danger branches, because knowing where a threat is makes a
real tactical sequence possible: get behind something, expose yourself enough to see,
wait, work around the flank, then go and look. It is also the clearest demonstration of the
engine's cover system, because four of the five actions search the same database with
**different evaluators** and get quite different behaviour out of it.

The three evaluators, named by what they optimize:

- **best cover** — maximum protection from the threat's direction. Used to hide.
- **close cover** — a position *near* the threat with a line on it. Used to look out.
- **angle cover** — a position at a different bearing to the threat than the creature's
  current one. Used to flank.
- **ambush cover** — a position covering the route between the threat and where the creature
  was. Used to wait for whatever is coming.

Every one of them is asked with the same two-radius fallback: try within ten units, and if
nothing qualifies, widen to thirty. That pattern is worth naming once — near cover is what
the behaviour wants, far cover is what stops the creature having no plan.

## State

Only the look-out action holds anything: a private random source, seeded from the
processor's cycle counter when the action object is built, used to decide crouch or stand.

**Notes** — the private, cycle-counter-seeded source is there so that this one choice is
independent of the global random stream. It matters because every creature in a squad
builds its actions at the same moment and would otherwise draw from the same sequence in
lockstep, producing six stalkers who all crouch or all stand. A rebuild needs
per-creature-independent randomness here, however it gets it.

## `DangerInDirectionTakeCover`

**Contract** — move to the best cover against the threat's position, facing the threat the
whole way, weapon aimed and ready with a burst shape appropriate to the range. Reports
`InCover` once the path completes.

```text
FUNCTION initialize()
  base.initialize()
  movement.mental_state := danger
  movement.body_state   := standing
  movement.path_type    := level path, smooth detail
  movement.gait         := run or walk, chosen at random
  weapon_goal(AIM_READY, best_weapon, burst shape for the current range)

FUNCTION execute()
  base.execute()
  IF no danger selected THEN RETURN
  threat := selected danger position
  sight  := watch the threat position

  point := best_cover(near = my position, radius = 10, evaluator = best,
                      threat, standoff 10, arc 170, spread 10)
  IF point is none THEN retry with radius 30
  IF point exists THEN steer to it ELSE head for nearest accessible position
  IF path completed THEN set property InCover = true
```

**Invariants** — the eyes are on the threat from the first cycle, not on the destination.
This is the whole difference from the unknown-danger take-cover: there, the creature does
not know where to look and watches its path; here it backs toward cover keeping the threat
in view.

**Notes** — the run-or-walk coin flip is per invocation. A squad reacting to the same shot
does not move as a block.

The action's declared sight-preference flag is never read; it is the residue of the
unknown-danger version this was copied from. So is a commented-out block that would have
set the brain's cover-affecting flag depending on whether the creature actually reached its
point. A rebuild should drop both.

## `DangerInDirectionLookOut`

**Contract** — move from hiding to a position that can *see* the threat, crouched or
standing by a per-creature coin flip, and report `LookedOut` when there. Also reports
success immediately if the creature's current position already has enough cover.

```text
FUNCTION initialize()
  base.initialize()
  set property UseCrouchToLookOut := private_random_boolean()
  movement.path_type    := level path, smooth detail
  movement.mental_state := danger
  movement.body_state   := crouch if UseCrouchToLookOut else standing
  movement.gait         := walk
  movement.head_for_nearest_accessible_position()
  weapon_goal(AIM_READY, best_weapon, burst shape for the current range)
  inertia_time := 1 second

FUNCTION execute()
  base.execute()
  IF no danger selected THEN RETURN
  threat := selected danger position
  sight  := watch the threat position

  IF cover_value_at(my position) >= 3 THEN
    stay where I am
    set property LookedOut = true            # already well enough placed
    RETURN

  point := best_cover(near = my position, radius = 10, evaluator = close,
                      threat, standoff 10, arc 170, spread 10)
  IF point is none
     OR (point is where I already stand AND path completed) THEN
    retry with radius 30
  IF point exists THEN steer to it ELSE head for nearest accessible position

  IF point exists AND I am within half a unit of it AND path completed THEN
    set property LookedOut = true
    stop
```

**Invariants** — the crouch choice is written into the planner's storage rather than kept
in the action, because the *hold-position* action that follows reads it to keep the same
posture. Posture continuity across two actions is what makes the pair read as one
behaviour.

**Notes** — the early-out on the creature's current cover value is the important line. Cover
is a precomputed per-navigation-vertex measure of exposure, and a value of three or more
means the creature is already behind something substantial. Rather than walk to a nominally
better point, it declares itself looked-out and stays. Without this, a creature in good
cover would shuffle every time the branch ran.

The "point is where I already stand, and I have stopped" condition forcing a wider search
is a stall breaker: the near search can legitimately return the creature's own position,
and re-steering to it would leave the creature permanently arrived and permanently not
looked out.

Half a unit is the arrival tolerance, tighter than the one-unit tolerance the cover
*stability* logic uses elsewhere, because this test decides a behavioural transition rather
than whether to re-plan.

## `DangerInDirectionHoldPosition`

**Contract** — stand or crouch in place, weapon up and aimed at the threat, for five to ten
seconds; then, **if the squad permits it**, declare the position held and release the cover
claim so the flanking step can begin.

```text
FUNCTION initialize()
  base.initialize()
  movement.path_type    := level path, smooth detail
  movement.head_for_nearest_accessible_position()
  movement.mental_state := danger
  movement.body_state   := the posture chosen by look-out
  movement.gait         := stand still
  weapon_goal(AIM_READY, best_weapon, burst shape for the current range)
  inertia_time := 5 seconds + uniform random up to 5 more

FUNCTION execute()
  base.execute()
  IF no danger selected THEN RETURN
  threat := selected danger position
  IF cover_value_at(my position) < 3 THEN set property LookedOut = false
  sight := watch the threat position
  IF inertia elapsed AND squad.may_detour() THEN
    set property PositionHolded = true
    set property InCover        = false
  re-issue weapon_goal(AIM_READY, best_weapon, burst shape for the current range)
```

**Invariants** — the transition out is gated on a **squad-level permission**, not only on
the timer. The squad coordinator allows only a limited number of members to be flanking at
once; the rest keep holding. That is how a group produces the appearance of fire-and-
manoeuvre without any member knowing what the others are doing.

Clearing `InCover` alongside setting `PositionHolded` is what forces the plan forward:
without it the creature would satisfy the flanking action's precondition while still
believing it was in cover, and the branch could loop.

**Notes** — the per-cycle cover check retracts `LookedOut` if the creature has drifted
somewhere exposed. That is the branch's only self-correction: a creature pushed out of its
position by physics or by an ally goes back to the look-out step rather than holding an
untenable spot.

The burst shape is recomputed and re-issued every cycle because the threat can move, and a
burst shape chosen for a distant target is wrong for one that has closed.

## `DangerInDirectionDetour`

**Contract** — work around to a position at a different bearing to the threat, then report
the flank complete. Requires the threat to be an actual remembered object, not just a
position, because the evaluator needs the threat's navigation vertex.

```text
FUNCTION initialize()
  base.initialize()
  tell the squad I am flanking
  movement.path_type    := level path, smooth detail
  movement.body_state   := standing
  movement.gait         := walk
  movement.mental_state := danger
  weapon_goal(AIM_READY, best_weapon, burst shape for the current range)
  release my cover claim

FUNCTION execute()
  base.execute()
  IF the danger has no associated object THEN RETURN
  remembered := my memory of that object
  IF I do not remember it THEN RETURN

  IF path completed THEN
    point := best_cover(near = my position, radius = 10, evaluator = angle,
                        threat position, standoff 10, my weapon's range,
                        the threat's navigation vertex)
    IF point is none THEN retry with radius 30
    IF point exists THEN steer to it ELSE head for nearest accessible position
    IF path completed THEN set property EnemyDetoured = true

  sight := watch the remembered position of the threat
```

**Invariants** — the search only runs when the previous path has completed. The flank is a
*chain* of short moves, each picked when the last finished, rather than one long route.
That is what makes a flanking stalker move in bounds from cover to cover instead of running
a smooth arc, and it is also what lets the manoeuvre abort cleanly at any bound.

The completion test is the same `path_completed` consulted twice in one cycle: once to
decide whether to pick a new point, once — after the pick — to detect that no new point was
needed. When the evaluator returns the creature's current position the two are both true and
the flank ends.

**Notes** — the angle evaluator is parametrized with the creature's own **weapon range**,
which is what ties the flank's geometry to what the creature is carrying: a shotgun user
flanks close, a sniper flanks wide.

Announcing the flank to the squad at entry is the other half of the permission the
hold-position action asked for. The squad counts flankers; releasing the cover claim at the
same moment frees the point for whoever is still holding.

## `DangerInDirectionSearch`

**Contract** — the branch terminator. Move to a position covering the route between the
threat and where the creature originally saw it from, wait there, and then stop believing
in the threat.

```text
FUNCTION execute()
  base.execute()
  IF the danger has no associated object THEN RETURN
  remembered := my memory of that object
  IF I do not remember it THEN RETURN

  IF path completed THEN
    point := best_cover(near = threat position, radius = 10, evaluator = ambush,
                        threat position, where I was when I saw it, standoff 10)
    IF point is none THEN retry with radius 30
    IF point exists THEN steer to it ELSE head for nearest accessible position

    IF path completed AND inertia elapsed THEN
      IF the danger still names an object THEN
        stop remembering that object
      ELSE
        stamp the danger memory's time line to now

  sight := watch the remembered position of the threat
```

**Invariants** — the ambush search is anchored at the **threat's** position, not the
creature's: the creature is looking for a place that covers the threat, which is generally
near the threat rather than near itself. Every other search in this file anchors on the
creature. The difference is what turns "search" from a retreat into an advance.

**Notes** — the two ways of ending are not equivalent, and the choice between them is the
most consequential line in the branch. Forgetting the *specific object* leaves every other
danger the creature knows about intact, so a stalker that searched out one shooter still
reacts to a second. Stamping the whole danger time line retires *everything* older than
now, and is used only when there is no object to forget — a danger that was only ever a
position.

The action is also the point at which a fruitless search ends. If the threat is never found,
the creature ends up at an ambush position with its weapon on the approach, and stops
reacting. That is a better resting state than returning to where it started, and it is why
this branch terminates by moving rather than by standing still like the grenade one.
