# src/xrGame/smart_cover_loophole_planner_actions.h

> Declares what a creature actually does at a loophole — idle, look out, fire, reload — and the four posture transitions between them.

**Needs** — [`smart_cover_planner_actions.h`](smart_cover_planner_actions.h.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`xrServer_Space.h`](../xrServerEntities/xrServer_Space.h.md)
**Used by** — [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md) · [`smart_cover_loophole_planner_actions_inline.h`](smart_cover_loophole_planner_actions_inline.h.md)
**Tier floor** — T2: planner operators that drive aiming and weapon goals

## Purpose

Declares the surface implemented in
[`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md).

Where [`smart_cover_planner_actions.h`](smart_cover_planner_actions.h.md) declares the
operators that *move* a creature, this declares the ones that *occupy* it. The inheritance
chain is the design: every one of them needs to decide where the creature looks while its
animation plays, and that decision — five cases, in a fixed priority — lives once in the
base.

## `loophole_action_base`

**Contract** — supplies `setup_sight`, the shared aiming decision, to every loophole
action. Its private helpers are the five cases; only the entry point is visible to
subclasses. See the implementation for the priority order, which is load-bearing.

## `loophole_action`

**Contract** — an action named by its planner operator name, which is *also* the authored
action name at the loophole. On entry it picks one clip at random from the loophole's list
for that action's idle purpose, and plays it. Holds its own random stream.

## `loophole_action_no_sight`

**Contract** — a loophole action that pins aiming to the animation's own direction instead
of aiming at anything. Used for idle and, through it, reload.

## `loophole_lookout`

**Contract** — the peek. Arms the head-bone aim solver against the *start* of its clip and
re-evaluates the aiming decision every cycle.

## `loophole_fire`

**Contract** — firing. Arms the weapon-bone aim solver, re-picks between its idle and shoot
clips each cycle depending on whether the head has arrived and the shot makes sense, and
issues the weapon goals on the clip's marker.

## `loophole_reload`

**Contract** — a no-sight loophole action that additionally commands the weapon to a
forced full aim, which is what makes the reload animation and the weapon state agree.

## `transition`

**Contract** — the base for the four posture changes. Picks a clip from the loophole's
*inner* transition graph between two named actions, and on the clip's end flips the two
readiness properties it was constructed with.

## `idle_2_fire_transition` / `fire_2_idle_transition` / `idle_2_lookout_transition` / `lookout_2_idle_transition`

**Contract** — the four concrete transitions. They differ only in which aim solver they
arm, which end of the clip they match, whether they arm forward or backward blend
callbacks, and whether they suspend aiming for the duration. Those four axes are the whole
content; see the implementation.

## Notes

`idle_2_fire_transition` takes a use-weapon flag and ignores it. The animation planner
passes true at both of its construction sites, so the intended alternative was never
exercised; a rebuild should drop the parameter.
