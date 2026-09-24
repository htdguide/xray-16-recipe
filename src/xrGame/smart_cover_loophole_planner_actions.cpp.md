# src/xrGame/smart_cover_loophole_planner_actions.cpp

> What a creature does while standing at a loophole: where it looks, which clip it plays, when it pulls the trigger, and how it changes posture.

**Needs** — [`smart_cover_loophole_planner_actions.h`](smart_cover_loophole_planner_actions.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_action.h`](smart_cover_action.h.md) · [`smart_cover_description.h`](smart_cover_description.h.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`object_handler_planner.h`](object_handler_planner.h.md) · [`Weapon.h`](Weapon.h.md) · [`property_storage.h`](property_storage.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-cycle aiming and weapon commands on the AI path

## Purpose

The nine operators here are what a player actually sees when a creature takes cover. Three
ideas run through them.

**Where to look is decided in one place, by priority.** A creature at a loophole may have
been told to watch a position, told to watch an entity, have an enemy, or have none of
those. One routine ranks those cases and sets the look order accordingly, and every action
calls it.

**The loophole's arc is a hard wall.** Whatever the creature wants to look at, if it is
outside the loophole's arc the creature looks at the *nearest edge of the arc* instead. It
never turns to face something the cover will not let it face, which is what stops a
creature in a firing slit from aiming through the wall.

**A posture change is an animation with a solver on one end.** Going from idle to firing
arms the aim solver against the clip's *end* pose; coming back arms it against the
*start*. That is what makes the weapon track the target smoothly across a transition
instead of snapping at the boundary.

## `setup_sight` — the aiming decision

**Contract** — sets the creature's look order for this cycle and reports whether it did.
Four cases in a fixed priority. The argument says whether to force the aiming update
through immediately rather than leaving it for the next frame.

```text
FUNCTION setup_sight(force_now) -> bool
  IF the loophole has an authored fire position    -> process_fire_position; RETURN true
  IF the loophole has an authored fire object      -> process_fire_object;   RETURN true
  IF there is no selected enemy                    -> process_default;       RETURN true
  RETURN process_enemy(force_now)
```

**Invariants** — authored targets outrank the creature's own enemy. A cover placed to watch
a doorway keeps watching the doorway even when the creature has an enemy elsewhere; the
level designer's intent wins over the creature's.

**Notes** — "force now" means the aiming manager's per-frame pass is run immediately with a
zero time step, which applies the new order's angles without advancing the smoothing. The
transitions pass true and the steady-state actions pass false: a transition must have the
creature already oriented when its clip starts, while an idling creature can turn over the
next few frames.

### `nearest_loophole_direction`

**Contract** — given a world position outside the loophole's arc, returns whichever arc
edge is closer to it.

```text
FUNCTION nearest_loophole_direction(position) -> vector
  facing = the loophole's world fov direction
  half   = half the loophole's arc
  left   = facing rotated by -half about the vertical
  right  = facing rotated by +half
  d      = normalize(position - the loophole's world eye point)
  RETURN whichever of left, right is more nearly aligned with d
```

**Invariants** — the rotation is applied to the *heading only*; the facing's pitch is
carried through unchanged. The arc is horizontal, so a loophole angled downward keeps its
downward angle at both edges.

### `process_fire_position` / `process_fire_object`

**Contract** — aim at an authored point or an authored entity. If it is inside the arc, a
look-at-position or look-at-entity order is issued with torso look on; if not, a
look-in-direction order at the nearest arc edge.

**Notes** — both are annotated in the original as the place a *loophole change* should be
considered: the right answer to "my target is outside this loophole's arc" is to move to a
loophole that covers it, not to stare at the edge. That work was never done. A rebuild that
does it changes cover behaviour substantially for the better, and should treat the arc-edge
look as the fallback when no other loophole covers the target either.

### `process_default`

**Contract** — with no target of any kind, the creature's look order is the
animation-driven one: the clip owns the head and the aiming manager only keeps its own
state consistent. Also marked as a place loophole selection should happen.

### `process_enemy`

**Contract** — aim at the selected enemy. Inside the arc: a look-at-entity order if the
enemy is *visible right now*, and a look-at-position order at the enemy's remembered
position if it is not. Outside the arc: the nearest arc edge.

**Invariants** — the visible/remembered split is what makes a creature in cover keep
covering where it last saw someone rather than tracking a target through a wall. The
remembered position comes from the creature's memory, which decays; the vocabulary is the
[feel](../../GLOSSARY.md) system's.

## `loophole_action` — the family base

**Contract** — on entry, picks one clip uniformly at random from the loophole's list for
this action's *idle* purpose, and plays that clip for the action's duration. The action's
name, as given by the planner, is also the authored action name at the loophole; that
coupling is how an operator finds its clips.

**Invariants** — the draw is from the action's own random stream, per creature, which again
keeps a squad from synchronizing.

## `loophole_action_no_sight`

**Contract** — as above, plus it pins the look order to the animation-driven one on entry
and never revisits it. Used for the idle operator: an idling creature's head is entirely
the clip's business.

## `loophole_reload`

**Contract** — a no-sight loophole action that, when naming its clip, additionally commands
the weapon subsystem to a *forced full aim* with the creature's best weapon.

**Notes** — the weapon command is issued from the clip-selection call rather than from
initialization, so it is re-issued at every clip boundary for as long as the reload runs.
That is harmless because the goal is idempotent, and it is what keeps the weapon in the
aimed state across a multi-clip reload.

## `loophole_lookout` — the peek

**Contract** — on entry, arms the **head** aim solver against the *start* of its clip; each
cycle, re-evaluates the aiming decision without forcing it through; on exit, disarms the
solver.

**Invariants** — matching the clip's *start* pose is right for the lookout because the peek
clip begins where the idle left the creature, so the solver's correction starts at zero and
grows. Matching the end would make the head jump at the first frame.

## `loophole_fire` — firing

**Contract** — the busiest action. Arms the **weapon** aim solver against its clip's start,
then each cycle decides between the loophole's idle and shoot clips, and issues the actual
weapon goals from the clip's marker hooks.

```text
FUNCTION initialize()
  the base initialization                        # picks an idle clip, sets the action id
  firing = true
  arm the weapon aim solver on this clip's start pose

FUNCTION execute()
  purpose = "idle"
  IF the head has arrived AND firing
     AND (the creature cannot kill its enemy OR firing makes sense)
    purpose = "shoot"
  ELSE
    firing = false
  clip = a uniform pick from the loophole's clips for (this action, purpose)
  setup_sight(force = false)

FUNCTION on_animation_end()   firing = NOT firing
FUNCTION finalize()           disarm the aim solver
```

**Invariants** —

- **The head must have arrived before the creature shoots.** The arrival test is the sight
  order's own ([`sight_action.cpp`](sight_action.cpp.md)); without it a creature would fire
  while still turning.
- The second half of the fire condition is a disjunction that reads oddly and is
  deliberate: firing is allowed when the creature *cannot* kill its enemy (suppressive
  fire, and the case where the enemy is not a valid kill target at all) **or** when the
  shot makes sense on its own terms. Only a creature that both can kill and for which the
  shot does not make sense holds fire.
- **`firing` toggles at every clip boundary**, so a creature alternates shoot and idle
  clips rather than firing continuously. That is the burst rhythm, and it is produced by a
  single flip rather than any timer.
- Falling out of the fire condition sets `firing` false, which the toggle then flips back
  true at the next boundary — so a creature that was blocked for one clip resumes on the
  next rather than latching off.

### `on_mark` — pulling the trigger

**Contract** — the clip's event marker has been crossed. Commands the weapon subsystem to
fire without reloading, for exactly a magazine's worth of rounds. Does nothing if the
creature has no weapon.

**Invariants** — the burst length is *the magazine size*, passed as both the minimum and
the maximum. The creature does not empty the magazine, because the goal is replaced at the
next marker or by the no-mark hook below; what the magazine size buys is an upper bound
large enough that the burst is ended by the animation, not by the count. A rebuild that
uses a small fixed count will make creatures fire in fixed bursts regardless of the clip.

### `on_no_mark` — between markers

**Contract** — on every query where the marker was *not* crossed, commands the weapon to
idle with a small round count. Skipped entirely when the cover does not permit firing.

**Invariants** — the pair of hooks is a latch: the marker starts a burst and every
subsequent query stops it. That is why `on_no_mark` exists at all, and why the base class
guarantees exactly one of the two fires per query.

## `transition` — the posture-change base

**Contract** — on entry, picks a clip uniformly at random from the loophole's *inner*
transition graph edge between the two named actions. On the clip's end, sets the source
readiness property false and the destination true in the planner's world state, and stamps
the transition time.

```text
FUNCTION initialize()
  clip = uniform pick from loophole.transition_animations(action_from, action_to)

FUNCTION on_animation_end()
  planner.state[state_from] = false
  planner.state[state_to]   = true
  planner.last_transition_time = now
```

**Invariants** — the property flip happens on the animation's **end**, not at the action's
start or finish. That is what makes the planner's readiness flags describe the creature's
*actual* posture: the creature is not ready to fire until the clip that raises its weapon
has finished playing. The whole operator table in
[`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) rests on this.

## The four concrete transitions

**Contract** — each differs from the others on four axes and nothing else.

```text
                       solver   clip end matched   blend callbacks   suspend aiming
idle -> fire           weapon   end                forward           no
fire -> idle           weapon   start              backward          yes
idle -> lookout        head     end                forward           no
lookout -> idle        head     start              backward          yes
```

Each arms its solver and its callbacks on entry, forces the aiming decision through
immediately, and on exit disarms the solver, removes the blend callbacks and restores the
ordinary bone callbacks.

**Invariants** —

- **Which end of the clip the solver matches is decided by which end of the transition the
  creature is aiming at.** Going *into* a firing or peeking posture, the pose that matters
  is the one the clip ends in — that is where the weapon must be pointing. Coming *out* of
  one, it is the pose the clip starts from. Getting this backwards makes the weapon swing
  wildly through the transition.
- **Forward versus backward blend callbacks** follow the same axis: entering a posture the
  correction is applied forward across the blend, leaving it, backward. The animation layer
  consumes this distinction directly.
- **Only the outbound transitions suspend aiming**, and they re-enable it in their own
  finalization. Suspending on the way out is what stops the aiming manager holding the
  creature's weapon on target while the clip is putting it down; on the way in there is
  nothing to hold yet.
- Every one of the four removes existing bone callbacks before installing its own and
  restores plain bone callbacks afterwards. Skipping either half leaves the creature with
  duplicated or missing animation events — the footstep and casing-eject markers.

**Notes** — the two inbound transitions contain a disabled suspend-aiming call, left
commented in the original. Enabling it would make entering a posture symmetric with leaving
it; it is disabled because suspending on the way in loses the target the creature was
already tracking, and the visible result is a creature that finishes its pop-up aimed at
nothing.
