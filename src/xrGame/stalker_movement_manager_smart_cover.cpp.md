# src/xrGame/stalker_movement_manager_smart_cover.cpp

> Getting a human into a piece of authored furniture and keeping them there: walking to the entry point, handing the body over to an animation, and the target the planner is driving toward inside.

**Needs** — [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md) · [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md) · [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md) · [`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_movement_params.h`](stalker_movement_params.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — reached through its declarations in [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md); callers name that, not this file.
**Tier floor** — T2: a state machine handing control between the pathfinder and the animation system

## Purpose

The lifecycle half of the smart-cover layer. Its two siblings own the routing between
loopholes ([`..._loopholes.cpp`](stalker_movement_manager_smart_cover_loopholes.cpp.md)) and
the visibility questions
([`..._fov_range.cpp`](stalker_movement_manager_smart_cover_fov_range.cpp.md)); this file
owns the **boundary**: the moment a creature stops being a thing on the navigation mesh and
becomes a thing inside a piece of furniture, and the moment it stops again.

The one idea a rebuilder needs: **inside a smart cover, the animation system owns the
creature's transform.** Outside it, the movement manager computes a position and the
animation follows. Inside, a named animation is played and the creature's position is
wherever that animation puts it. The handover in each direction is what this file is for,
and it is the source of every assertion in it.

## State

```text
RECORD SmartCoverLayer             # extends the obstacles layer
  current  : MovementParams        # where the creature is: cover, loophole, and the
                                   # fire target it is oriented on
  target   : MovementParams        # where it is being sent
  path     : list<text>            # the loophole route, in the routing sibling
  current_transition : optional<Transition>          # the transition being played
  animation_selector : AnimationSelector             # chooses animations while inside
  target_selector    : PlannerTargetSelector         # what the planner wants inside
  entering_with_animation : bool
  enter_cover_id, enter_loophole_id : text           # committed at the start of the entry
  enter_animation    : MotionID
  non_animated_loophole_change : bool
  default_behaviour  : bool
  apply_loophole_direction_distance : real
```

**Invariants**
- A creature that has a cover always has a loophole. Asserted on every update; a cover
  without a loophole is a state that cannot be acted on.
- The entry animation and the loophole it commits to are captured *before* the animation
  starts and are not re-derived while it plays, because the target may change mid-animation
  and the creature must still arrive where it set out for.
- `current` and `target` are never both describing the same loophole while a transition is
  in flight — that is the condition a transition exists to resolve.

## `update`

**Contract** — the per-frame step, run in place of the base manager's when a cover is
involved. Four states, distinguished by whether the creature has a cover and whether it is
being sent to one.

```text
FUNCTION update(time_delta)
  IF the creature is being destroyed THEN RETURN

  IF NOT in a cover
    IF NOT being sent to one
      base.update(time_delta) ; RETURN     # ordinary walking
    enter_smart_cover(time_delta) ; RETURN

  IF a loophole change without animation is pending
    perform it now
  IF the change took us out of the cover
    base.update(time_delta) ; RETURN

  IF being sent to the loophole we are already in
    adopt the target's fire object and fire position   # same place, new aim
  target_selector.update()
```

**Notes** — the last branch is the common case and the subtle one. A creature already in the
right loophole is not moving at all; what changes is *what it is oriented on*. Adopting the
target's fire object and fire position without any transition is how a creature in cover
swings from one enemy to another without leaving.

Once inside, the per-frame work is not movement — it is the **planner target selector**,
which decides which of the loophole's authored actions (idle, look out, fire, fire without
looking out) the creature should be performing. Movement inside a cover is entirely the
animation system's.

## `reach_enter_location`

**Contract** — walks the creature to the point the entry animation starts from, faces it the
right way, and when it has arrived and is holding the right weapon, starts the entry. This
is the longest routine in the file and it is a sequence of gates, each of which returns
early until it passes.

```text
FUNCTION reach_enter_location(time_delta)
  # walk on the level path with smoothing, carrying the target's posture over
  current.path_type = level_path ; current.detail_path_type = smooth
  current.mental_state, body_state, movement_type = the target's

  loophole = the target loophole IF it can be entered ELSE the nearest enterable one
  position = the cover's transform applied to the entry animation's start position
  vertex   = the navigation vertex containing that position

  IF the vertex or the position is outside the creature's permitted space
    # repair, in a strict order
    snap position onto the vertex — its centre if outside it, its ground plane if inside
    IF the position is still forbidden
      move both to the nearest permitted vertex
    ELSE IF only the vertex is forbidden
      move both to the nearest permitted vertex from the vertex's own position
  set the level destination to vertex ; set the desired position to position

  direction = the cover's entry direction for that loophole
  set the desired direction
  IF close enough to the target
    tell the sight system to face that direction explicitly

  base.update(current)

  IF the path is not complete            THEN RETURN
  IF the sight has not reached its target THEN RETURN

  IF this cover can be fired from
    item = what is in the hands
    IF nothing is held, or it is not in the primary weapon slot
      IF the equip goal has not been reached THEN RETURN
      set the goal: equip the best weapon ; RETURN

  hand the creature's transform to the animation system, aimed at (position, direction)

  IF this transition has no animation
    enter_smart_cover() ; RETURN          # step in without one

  tell the sight system to follow the animation's own direction
  disable hit reactions
  entering_with_animation = true
  commit enter_cover_id and enter_loophole_id from the target
  resolve the entry animation by name
  install this layer's own animation selector, end callback and speed modifier
```

**Invariants** — the gates are ordered by cost and by dependency: arrive, then face, then
arm, then commit. A creature that has arrived but is still turning must not start the
animation, because the animation assumes a starting orientation.

**Notes** — the *position repair* deserves its own reading. The entry point is authored as an
offset in the cover object's own frame, so it is wherever the object was placed — and nothing
guarantees the level designer put it somewhere the navigation mesh permits, or somewhere this
particular creature is allowed to go. Three successive repairs handle it: snap onto the
vertex, then relocate to the nearest permitted vertex if the position is forbidden, then the
same if only the vertex is. The order matters because each repair reads the result of the
previous. A rebuild that skips the repair will have creatures refuse to use authored covers
placed near restrictor boundaries, which is most of the interesting ones.

**Requiring a primary-slot weapon before entering a firing cover** is a gameplay rule, not a
technical one: a creature that climbs into a firing position holding a pistol or nothing
looks wrong, so it is sent to equip its best weapon and the entry waits. Note that it waits
by *returning* — the equip is a goal set on the object handler and the entry is retried next
frame.

Disabling hit reactions on entry is the physical counterpart of the handover: a hit-reaction
animation would fight the cover animation for the same skeleton and the creature would
visibly pop out of the furniture. It is re-enabled on exit, and this pairing is the only
thing that guarantees it.

## `enter_smart_cover`

**Contract** — completes the entry once the creature is in position, with or without an
animation having played. Chooses between the committed entry loophole and the target one,
binds the inside-the-cover animation selector, and initializes it.

```text
FUNCTION enter_smart_cover()
  loophole = the target loophole IF enterable ELSE the nearest enterable one
  bind the cover animation selector

  IF not yet in a cover AND an entry was committed
     AND the target has since changed to a different cover or loophole
    # honour the commitment: we are mid-way into *that* loophole
    current.cover = enter_cover_id ; current.loophole = enter_loophole_id
  ELSE
    advance to the next loophole on the route
    IF we arrived at the target loophole
      adopt the target's fire object and fire position
  animation_selector.initialize()
```

**Notes** — the commitment branch is the whole reason the entry identifiers are stored. The
entry animation takes time, and the target can change while it plays — a script retargets the
creature, an enemy moves. The creature nonetheless *ends up* in the loophole the animation
carried it into, because that is where its body now is. Overriding the commitment with the
new target would leave the creature's recorded position disagreeing with its actual one,
which inside a cover is unrecoverable.

## `select_animation` / `on_animation_end` / `modify_animation`

**Contract** — while the entry animation plays, this layer temporarily *is* the creature's
animation source. `select_animation` reports the committed entry animation and declares that
the animation controls the creature's transform. `on_animation_end` completes the entry — or,
if the target has since been cleared, releases the animation system and leaves the creature
standing where the animation put it.

`modify_animation` scales the playing blend's speed by a process-wide factor, in development
builds only, so the transitions can be slowed down and watched.

**Invariants** — declaring that the animation controls the transform is the handover itself.
From that moment the movement manager's computed position is ignored and the creature is
wherever the animation's root motion places it.

## `on_smart_cover_enter` / `on_smart_cover_exit`

**Contract** — the two ends of the handover, each doing exactly what the other undoes.
Entering disables hit-reaction animations. Exiting re-enables them, clears the transition
state and the pending loophole change, finalizes the animation selector, releases the
animation system, clears the cover identity and runs one ordinary movement update to put the
creature back on the navigation mesh.

**Notes** — the final ordinary update inside the exit is what re-establishes the creature as
a thing that walks. Without it, the creature would be out of the cover and have no path, no
destination and no posture until something else set one.

## `bind_global_selector` / `unbind_global_selector`

**Contract** — install or remove the cover animation selector as the creature's animation
source. Installing additionally aims the animation system's target transform at the current
loophole's field-of-view position and entry direction, so that the first animation selected
starts from the right place.

**Notes** — the selector is a *triple*: a chooser, an end callback and — in development
builds — a speed modifier. All three are installed and removed together, and removing them is
what returns the creature to its ordinary animation logic. Leaving one installed after an
exit is the failure mode this pairing guards against.

Aiming the target transform is skipped when there is no current cover, which is the case
during an entry animation: the creature is on its way in and its target transform was already
set by `reach_enter_location`.

## `loophole_path`

**Contract** — finds the route from one loophole to another through the cover's authored
transition graph. Wraps each endpoint in a marker distinguishing "entering at" from "leaving
by", searches the transition graph with no cost, iteration or vertex limits, and fails the
process if no route exists.

**Notes** — the endpoint markers are how the world outside the cover is represented in a
graph whose vertices are loopholes: entering is a route *from* the outside marker, leaving is
a route *to* it. One graph therefore expresses entry, internal movement and exit.

The search is deliberately unbounded. A cover's transition graph is a handful of nodes
authored by hand; a bound would be meaningless and a missing route is an authoring error,
which is why it is fatal rather than a refusal. The source flags the unbounded limits as
worth re-checking — they are expressed by casting an all-ones integer into the cost type,
which is a way of writing "no limit" that a rebuild should simply say directly.

## `exit_transition` / `current_transition` / `target_approached`

**Contract** — `exit_transition` reports whether the next step on the loophole route leaves
the cover, by testing the route's second entry against the outside marker.
`current_transition` refreshes the route and reports the transition now being played, failing
if there is none. `target_approached` reports whether the creature is within a given distance
of its destination, and is false whenever the path is not fresh — an unfresh path's distance
is meaningless.

**Notes** — `current_transition` asserts that the current and target loopholes differ. A
transition from a loophole to itself is not a thing that exists, and asking for one means a
caller has already gone wrong.

## The planner targets

**Contract** — five entry points scripts and the planner use to say what the creature should
do inside its cover: `target_idle`, `target_lookout`, `target_fire`,
`target_fire_no_lookout`, and `target_default`. Each sets a world-state property the planner
searches toward; the last sets a flag instead.

**Notes** — each of the four action setters carries a commented-out precondition block that
would have refused the request when the creature has no cover, or when the loophole's
authored action set does not include that action, or (for firing) when no enemy is in the
loophole's field of view. All four were disabled and the requests are now unconditional. The
consequence is that asking for an action a loophole cannot perform is accepted and resolved
later, by the animation selector finding nothing to play. **Unrecovered**: whether the checks
were removed because scripts depended on the requests being accepted, or because the checks
themselves were wrong.

`default_behaviour` is the fallback when no script has installed a target selector: the
creature behaves "by default" if it has a fire object or a fire position, and is otherwise
idle. With a script selector installed, the script's own flag wins.

## `cleanup_after_animation_selector` / `target_selector` / `in_smart_cover` / `remove_links`

**Contract** — `cleanup_after_animation_selector` invalidates the level and detail paths,
which is what a cover animation having moved the creature requires: the paths were computed
for where it used to be. `target_selector` installs a script callback as the inside-the-cover
decision maker. `in_smart_cover` reports true both for a creature inside one and for one
currently playing an entry animation — the state between the two is still "in a cover" as far
as every caller is concerned. `remove_links` clears the fire object from both the current and
the target parameters when that object is destroyed, in addition to the base layer's own
teardown.

## `on_frame` / `reinit` / construction

**Contract** — `on_frame` forwards unchanged and exists only so the layer has the hook.
`reinit` rebuilds the animation selector in place, constructing it on first use and
reconstructing it afterwards, and resets the target parameters. Construction creates the
target parameters and the planner target selector; the animation selector is deferred to
`reinit` because it needs the creature's property storage, which does not exist yet at
construction.

**Notes** — reconstructing the animation selector *in place* rather than replacing it is
done because the planner holds a reference to it that must stay valid across a
reinitialization. A rebuild that can re-seat the reference should simply replace the object.
