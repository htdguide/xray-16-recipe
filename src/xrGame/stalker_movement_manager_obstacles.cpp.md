# src/xrGame/stalker_movement_manager_obstacles.cpp

> What a walking human does about a world that moves: wait for a door, stop for someone in the way, replan around something that appeared, and give up quietly for a second when no path exists.

**Needs** — [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md) · [`stalker_movement_manager_space.h`](stalker_movement_manager_space.h.md) · [`restricted_object_obstacle.h`](restricted_object_obstacle.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`level_path_builder.h`](level_path_builder.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`doors_actor.h`](doors_actor.h.md) · [`doors_manager.h`](doors_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame obstacle queries plus a trial graph search under a mask

## Purpose

The per-frame half of obstacle avoidance. Its counterpart,
[`stalker_movement_manager_obstacles_path.cpp`](stalker_movement_manager_obstacles_path.cpp.md),
owns the path search; this file owns what happens between searches.

The whole of it is a cascade of four ways to not walk into something, ordered from cheapest
and most polite to most expensive:

1. **Wait for a door.** Something ahead is a door that is closed or moving. Stand still, at
   commanded speed zero, until it is not.
2. **Back off after a failure.** A search failed within the last second. Do not try again;
   walk the stale path.
3. **Stop for a dynamic obstacle.** Something is in the way right now. Stand still; it will
   probably move.
4. **Replan.** A static obstacle has changed the mesh. Rebuild the path immediately, inline.

Each step falls through to the base manager's ordinary motion when it does not apply.

## State

Declared in
[`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md).

## Construction / `Load` / `create_restricted_object`

**Contract** — creates the creature's identity in the door system and constructs the two
avoiders, each handed a reference to the same shared failure flag. `Load` additionally
**turns off the path builder's delay-after-failure**.

**Notes** — the shared failure flag is how the avoiders and the path search communicate
without knowing about each other: an avoider sets it when its trial search fails, and the
search loop reads it as "stop trying".

Turning off the builder's back-off is deliberate and is replaced by this layer's own
one-second back-off. The generic builder's delay is tuned for paths that fail because the
destination is unreachable; a path that fails because something is standing in a doorway
should be retried as soon as that changes, and the door system's own state gives a better
signal than a timer.

`create_restricted_object` supplies a restrictor that consults the two avoiders' active
queries in addition to the ordinary permitted and forbidden volumes. That is the hook by
which an obstacle becomes part of the creature's permitted space.

## `move_along_path`

**Contract** — the per-frame entry. Runs the cascade and then hands off to the base manager.
Never returns without the base manager having been given its chance to move the creature.

```text
FUNCTION move_along_path(movement_control, out dest_position, time_delta)
  IF the door system says this creature must wait
    # commanded speed zero, but still call through: the animation and the
    # physics character must keep being driven, or the creature freezes mid-stride
    remembered = commanded speed ; commanded speed = 0
    base.move_along_path(...)
    commanded speed = remembered
    RETURN

  IF obstacle avoidance is disabled          # development switch
    base.move_along_path(...) ; RETURN

  IF now < last_fail_time + fail_back_off
    base.move_along_path(...) ; RETURN       # a search just failed; walk the stale path

  IF the base manager has nothing to walk
    base.move_along_path(...) ; RETURN

  move_along_path_impl(movement_control, dest_position, time_delta)
```

**Invariants** — every branch calls through to the base manager. Zeroing the commanded speed
and still calling is the important pattern: the creature's animation, its physics character
and its facing all come from that call, so skipping it would leave a creature frozen in
whatever pose it had rather than standing still.

**Notes** — the back-off is **one second**. It exists because a failed search is expensive
and a world that blocked a path a moment ago will usually still block it. It is a feel value.

The door wait is checked *first*, before the avoidance switch and before the back-off,
because a closed door is not an obstacle to plan around — it is one to wait at. Planning a
route around a door the creature is about to open would send it the long way round every
time.

## `move_along_path_impl`

**Contract** — runs the two avoiders and decides between stopping and replanning.

```text
FUNCTION move_along_path_impl(movement_control, out dest_position, time_delta)
  dynamic_obstacles.update()
  IF dynamic_obstacles say movement is blocked
    stop (commanded speed zero) and call through ; RETURN

  static_obstacles.update()
  IF static_obstacles OR dynamic_obstacles need the path rebuilt
    rebuild_path()
  base.move_along_path(...)
```

**Notes** — the asymmetry between the two avoiders is the content. A **dynamic** obstacle —
another creature walking across the doorway — gets the creature to *stop*, because in a
moment it will be gone and a replanned path would be wasted and would look like panic. A
**static** obstacle — something that has come to rest in the way — gets a *replan*, because
waiting for it would be waiting forever.

The dynamic avoider is consulted before the static one is even updated. That ordering is a
cost decision: the dynamic check is cheap and short-circuits the expensive one.

## `rebuild_path`

**Contract** — rebuilds the level path and the detail path, **both inline, this frame**.
Invalidates the level path and then runs the pipeline twice, forcing each pass to complete
rather than deferring to a worker.

```text
FUNCTION rebuild_path()
  level_path.invalidate()
  force_inline_build() ; update_path()      # the level search
  force_inline_build() ; update_path()      # the detail smoothing
```

**Notes** — two calls, because the pipeline advances one stage per call and each must be
forced separately. Forcing inline is the point: an obstacle-triggered rebuild is a *reaction
to something the creature has just walked into*, and deferring it to a worker means one or
more frames of continuing to walk into it.

## `apply_border` / `remove_border`

**Contract** — marks every navigation vertex an obstacle covers as forbidden, excluding the
creature's own vertex and its destination, and additionally installs a search border between
those two. `remove_border` undoes both. Always paired.

```text
FUNCTION apply_border(query)
  start = the creature's current vertex ; dest = the path's destination
  restrictor.add_border(start, dest)
  FOR EACH vertex IN query.area
    IF vertex IS start OR vertex IS dest THEN CONTINUE
    mark vertex forbidden on the shared navigation mesh

FUNCTION remove_border(query)
  restrictor.remove_border()
  FOR EACH vertex IN query.area
    clear the mark
```

**Invariants** — the creature's own vertex and its destination are **never** marked. A
creature standing inside an obstacle's area — which happens constantly, since creatures are
themselves obstacles to each other — would otherwise have no vertex to start from and every
search would fail. Excluding the destination is the same reasoning at the other end.

The marks are set on the **shared, process-wide navigation mesh**, without a per-vertex
check that they were not already set. That is why the pairing is absolute: an unbalanced
apply leaves vertices forbidden for every creature on the level, and there is nothing that
would ever clear them.

## `can_build_restricted_path`

**Contract** — would a path to the destination still exist if this obstacle's area were
forbidden. Applies the border, runs a trial search into scratch storage, removes the border,
and reports the result. Records the failure in the shared flag. Leaves the creature's real
path untouched.

```text
FUNCTION can_build_restricted_path(query) -> bool
  apply_border(query)
  failed = NOT search(navigation mesh, current vertex, destination,
                      into scratch, with a 4096-vertex iteration ceiling)
  remove_border(query)
  RETURN NOT failed
```

**Notes** — the trial search is capped at **4096 expanded vertices**. This is asked
repeatedly as obstacles are considered one at a time, so an uncapped search on a large level
would dominate the frame. The cap means the answer is "no path within a reasonable search",
not "no path" — a creature will decline to route around an obstacle if doing so needs a very
long detour, and will instead stop or wait. That is the right behaviour and it is a
consequence of the cap rather than an explicit rule. 4096 is not derived anywhere.

## `is_going_through`

**Contract** — does this creature's path cross a given segment within a given distance
ahead, and if so how far along that segment. Returns the smallest crossing distance found,
or a negative marker for no crossing. Pure.

```text
FUNCTION is_going_through(frame, offset, max_distance) -> real
  IF the path is not actual, or empty, or already at its last point
    RETURN no crossing
  segment = (frame.origin, frame applied to offset)      # the query segment, in world space

  travelled = 0 ; best = none
  FOR EACH consecutive pair (previous, current) IN the path from the next travel point on
    d = crossing_distance(previous, current, segment, safe_width)
    IF d is a crossing THEN best = min(best, d)
    travelled = travelled + distance(previous, current)
    IF travelled > max_distance THEN BREAK
  RETURN best OR no crossing
```

The crossing test itself is two-dimensional — the horizontal plane only — and has three
outcomes:

```text
FUNCTION crossing_distance(a0, a1, b0, b1, safe_width) -> real
  CASE the two segments are the same line
    RETURN 0                        # definitely crossing
  CASE the two segments are parallel
    # not a crossing, but they may still be too close to pass each other
    RETURN 0 IF the perpendicular separation is under safe_width ELSE no crossing
  CASE they intersect
    RETURN distance from a0 to the intersection point
```

**Notes** — this is how a door decides whether a creature is about to walk through it, and
how one creature decides whether another is about to cross its path. The answer is a
*distance along the asker's own segment*, not a yes/no, so the caller can decide whether the
crossing is soon enough to matter.

The parallel case is the interesting one. Two paths that never intersect can still be too
close to walk side by side; the **0.35 metre** safe width is roughly a person's shoulders and
treats a near-parallel pass as a crossing. Not derived; a body-width feel value.

Everything is tested in the horizontal plane, ignoring height. Two creatures on different
floors of a building will read as crossing. The navigation mesh's own structure keeps this
from mattering in the shipped levels, and a rebuild on taller geometry would need the height
check.

## `prediction_speed`

**Contract** — overrides the base manager's prediction speed with **the animation's target
speed** rather than the movement manager's commanded speed.

**Notes** — this is a real correction. A human's actual velocity comes from the animation
that is playing, not from what the movement layer asked for; the two differ whenever an
animation is blending in or out. Anything predicting where this creature will be — including
the obstacle checks above and every shooter aiming at them — gets the animation's number.

## `remove_links` / `on_death`

**Contract** — `remove_links` forwards the "this object is going away" notification to both
avoiders as well as to the base manager; an avoider holding a reference to a destroyed
obstacle would otherwise mark vertices around nothing. `on_death` destroys the creature's
door-system identity, because a corpse neither opens doors nor waits for them.

**Notes** — `on_death` destroys the door actor while the rest of the movement manager lives
on, and every per-frame entry asserts the actor exists. A dead creature whose movement is
still being driven would fail that assertion. It does not happen because death stops the
movement driver first — an ordering that is a fact about the caller, not enforced here.
