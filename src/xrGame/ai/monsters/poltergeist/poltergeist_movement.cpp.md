# src/xrGame/ai/monsters/poltergeist/poltergeist_movement.cpp

> Advances a hidden poltergeist along its route by moving a position it carries itself, because while hidden it has no physical body for the shared follower to move.

**Needs** — [`poltergeist_movement.h`](poltergeist_movement.h.md) · [`poltergeist.h`](poltergeist.h.md) · [`../../../detail_path_manager.h`](../../../detail_path_manager.h.md)
**Used by** — [`poltergeist_movement.h`](poltergeist_movement.h.md)
**Tier floor** — T2: replaces the step where a route position is handed to the character controller

## Purpose

The shared path follower advances a creature by asking the character controller to move a
physical body and then reading back where it ended up. A hidden poltergeist has no body — see
[`poltergeist.cpp`](poltergeist.cpp.md) — so that loop has nothing to move.

This override closes the gap: while hidden, the creature's **graph position** is what advances
along the route, integrated directly from the desired speed and the elapsed time, with no
physics, no collision and no possibility of being blocked. The rendered position is that graph
position lifted by the drift height, and it is what everything outside the navigation layer
sees.

Two positions for one creature is the whole idea, and it is the reason a poltergeist can drift
through a doorway at head height while the navigation graph believes it is walking the floor.

## `move_along_path`

**Contract** — advance along the current detailed route and report where the creature now is.
While visible, defers wholly to the shared implementation. While hidden, runs the integration
below and writes both positions on the creature. Sets the follower's reported speed. Allocates
nothing.

```text
FUNCTION move_along_path(controller, out destination, elapsed) 
  IF NOT creature.hidden
    RETURN shared_implementation(controller, destination, elapsed)

  destination = creature.graph_position

  # --- nothing to advance along? come to rest -----------------------------
  IF the follower is disabled, or the route is finished, or the route is empty,
     or the route is already completed at this position,
     or we are on the route's last segment,
     or the desired speed is zero,
     or elapsed is negligible
    speed = 0
    destination = rendered_position()
    RETURN

  desired  = the follower's desired speed
  budget   = desired * elapsed          # how far we may travel this step
  remaining = budget

  # --- resync which segment we are on -------------------------------------
  WHILE not on the last-but-one segment
    IF we are further from the current waypoint than that waypoint is from the next,
       AND further from the current than from the next
      advance to the next segment              # we have overshot it
    ELSE
      BREAK

  target      = the next waypoint
  to_target   = target - destination
  distance    = |to_target|

  # --- consume the budget across as many segments as it reaches -----------
  WHILE remaining > distance
    destination = target
    IF there is no segment after this one THEN BREAK
    remaining = remaining - distance
    advance to the next segment
    IF there is no segment after that one THEN BREAK
    target    = the next waypoint
    to_target = target - destination
    distance  = |to_target|

  IF the segment index changed THEN notify the follower

  IF distance is negligible                     # standing on the final waypoint
    mark the route finished
    speed = 0
    destination = rendered_position()
    RETURN

  destination = destination + to_target scaled to remaining

  # --- report a smoothed speed --------------------------------------------
  travelled  = |that last step| + budget - remaining
  actual     = travelled / elapsed
  speed      = (desired + actual) / 2

  creature.graph_position = destination
  creature.position       = rendered_position()
  destination             = creature.position
```

**Invariants** — the graph position is *always* the thing advanced; the rendered position is
always derived from it and never the other way round. The route index only ever moves forward.
The creature cannot be blocked: there is no collision test anywhere in this routine, which is
precisely what "it floats through things" means mechanically.

**Notes** — the resync loop before the main advance exists because the graph position can drift
out of step with the route index when the route is rebuilt underneath the follower; it walks the
index forward while the creature is demonstrably past the current waypoint. Without it a rebuilt
route makes the creature double back.

The reported speed is the *average* of the desired speed and the speed actually achieved, not
the achieved one. That halves the response of whatever reads it — the animation blending — so
the floating animation does not snap when the route turns a corner. It is a smoothing choice,
not a measurement.

Every early exit reports the *rendered* position rather than the graph position, so a
stationary hidden poltergeist still reads as being at its drift height. Getting this wrong
drops the creature to the floor whenever it stops.

The route-completion checks are a chain of six conditions rather than one, because the shared
follower exposes each of them separately; a rebuild should collapse them into a single "is
there anywhere to go" question on the route object.

## `rendered_position`

**Contract** — the graph position lifted by the creature's current drift height. Pure.
