# src/xrGame/movement_manager_physic.cpp

> Turns the finished detail path into an actual position each frame, by handing a velocity to the physics character when anything is nearby and by teleporting the character when nothing is.

**Needs** — [`movement_manager.h`](movement_manager.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`Level.h`](Level.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`steering_behaviour.h`](steering_behaviour.h.md) · [`xrPhysics/IColisiondamageInfo.h`](../xrPhysics/IColisiondamageInfo.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame vector arithmetic against a physics character; no device or format contact, but it runs for every creature every frame

## Purpose

The end of the movement pipeline. Everything upstream produces a list of travel points;
this file walks a creature along that list at its commanded speed, decides on each frame
whether the physics solver needs to be involved at all, and converts collisions suffered
while walking into damage.

Its central decision — and the one a rebuild most needs to copy — is that **a creature
surrounded by nothing is moved by setting its position directly, with its physics character
switched off**. Only when something is within a couple of metres does the creature become a
simulated body with a velocity. This is what lets the world hold hundreds of walking
creatures at sixty frames per second.

## State

`Stateless` — it reads and writes the movement manager's fields and the detail path
manager's current travel point.

## Constants

```text
nearby_radius = 2.0 real   # metres; the radius searched for other physics objects.
                           # widened by 0.5 while the character is already enabled,
                           # so that a creature that has become physical does not
                           # flicker back to non-physical at the boundary
```

The half-metre widening is hysteresis, not tuning: without it a creature standing exactly
at the radius alternates between the two motion regimes every frame and jitters.

## `move_along_path` (the per-frame mover)

**Contract** — given the physics character, an output position and the frame's time step,
advance the creature along the detail path. Writes the resulting world position to the
output, updates the achieved speed, may fire the travel-point-change notification, may
apply a hit to the creature, and may enable or disable its physics character. Does not
allocate beyond reusing a scratch list of nearby objects. Runs on the frame thread.

```text
FUNCTION move_along_path(character, OUT position, time_delta)
  position := creature position

  IF NOT should_move()
    achieved_speed := 0
    IF the character is currently enabled
      let the physics character settle with zero desired speed
      read the position back from it
    apply_collision_hit(character)         # standing still can still be run into
    RETURN

  IF the character does not exist THEN RETURN         # e.g. wounded, animation-driven
  IF time_delta below epsilon THEN RETURN

  # 1. advance along the path geometrically
  travel_point := detail path's current travel point
  position := path_position(commanded_speed, creature position, time_delta,
                            travel_point, OUT remaining, OUT dist_to_target, OUT dir_to_target)
  IF travel_point changed THEN notify travel-point change
  store travel_point back into the detail path

  IF dist_to_target below epsilon                      # degenerate: we are on a point
    advance the stored travel point by one, clamped to the last
    achieved_speed := 0
    RETURN

  # 2. decide the motion regime
  nearby := physics objects within nearby_radius (+0.5 if the character is enabled),
            excluding ourselves
  motion := dir_to_target scaled to length (remaining / dist_to_target)
  position := position + motion

  velocity := normalized dir_to_target, with its vertical component clamped to ±0.8,
              renormalized, scaled by the commanded speed
  IF the creature is not in physics-only mode THEN give the character that velocity

  IF nearby is non-empty
    IF the character cannot be placed at the computed position
      read the character's own position instead
      let the physics solver advance it along the path at the commanded speed
      apply_collision_hit(character)
    ELSE
      mark the character's position as exact
    read the position back from the character
  ELSE
    place the character at the computed position, switch it off,
      mark its position as exact

  # 3. report the speed actually achieved
  real_speed := (length(motion) + commanded_speed*time_delta - remaining) / time_delta
  achieved_speed := 0.5*commanded_speed + 0.5*real_speed

  IF the detail path is now consumed AND we are not in physics-only mode
    zero the character's velocity and the achieved speed
```

**Invariants** — the achieved speed is a fifty-fifty blend of commanded and measured, not
the measured value. Animation selection reads this speed, and a raw measurement makes the
walk cycle stutter whenever the solver pushes the creature around; the blend is a
one-pole filter with the frame as its time constant.

The vertical clamp on the velocity direction (±0.8 of a unit vector, then renormalized)
caps how steeply a creature may drive itself. Without it, a path leg that points almost
straight up or down — which the navigation mesh does produce at ledges — turns into a
launch or a dive. The source notes the direction is not normalized on arrival, which is
why it is normalized twice.

**Notes** — the "cannot be placed" path is a *swept* test: the character is asked whether
it may occupy the computed position, and only if it refuses does the full solver run. So
the expensive path is taken only on an actual obstruction, not merely on proximity.

A commented-out steering-behaviour block sits in the middle of this function. The steering
manager exists and is constructed, but nothing here consumes it: the intended design was to
add an avoidance acceleration to the target before computing the direction. A rebuild
implementing local avoidance should add it at exactly that point.

## `should_move` (the six-way gate)

**Contract** — answers whether path following should run at all this frame. All six
conditions must hold; any one failing means the creature stands still.

```text
FUNCTION should_move() -> bool
  RETURN movement enabled
     AND the path is actual
     AND the detail path is non-empty
     AND the detail path is not already completed at our position
     AND the current travel point is not the last one
     AND the commanded speed is non-zero
```

**Notes** — "path completed" is asked with the strict, patrol-style flag here regardless of
how the path was built, so a creature stops at the final point rather than near it.

## `path_position` (walking the point list)

**Contract** — pure geometry over the detail path: given a speed, a start position and a
time step, return the position reached, and report through outputs how much of the
step's distance remains unconsumed, the distance to the next point and the direction to
it. Advances the caller's travel-point index. Does not touch the creature or the physics
world.

```text
FUNCTION path_position(velocity, position, time_delta,
                       INOUT travel_point, OUT remaining, OUT dist_to_target, OUT dir_to_target)
  remaining := velocity * time_delta

  # resynchronize the index: if we are further from the current point than that point is
  # from the next one, AND closer to the next one, we have already passed the current one
  WHILE travel_point < last index - 1
    IF distance(position, point[travel_point]) > distance(point[travel_point], point[travel_point+1])
       AND distance(position, point[travel_point]) > distance(position, point[travel_point+1])
      travel_point := travel_point + 1
    ELSE
      BREAK

  target := point[travel_point + 1]
  dir_to_target := target - position
  dist_to_target := length(dir_to_target)

  # consume whole legs while the step outlasts them
  WHILE remaining > dist_to_target
    position := target
    remaining := remaining - dist_to_target
    IF there is no point after travel_point+1
      RETURN position                       # ran off the end of the path
    travel_point := travel_point + 1
    IF there is no point after the new travel_point
      remaining := 0
      RETURN position
    target := point[travel_point + 1]
    dir_to_target := target - position
    dist_to_target := length(dir_to_target)

  RETURN position                            # remaining <= dist_to_target on exit
```

**Invariants** — on return, the unconsumed distance never exceeds the distance to the next
point, which is what lets the caller finish the step with a single scaled vector instead of
another loop.

The index resynchronization is not an optimization: the creature is pushed around by
physics and by scripted teleports, so the stored travel point can be arbitrarily wrong.
Both conditions are needed — being far from the current point alone happens legitimately
at the start of a long leg, and being nearer the next point alone happens on a sharp
corner.

**Notes** — a two-argument form of the same name answers "where would this creature be in
*t* seconds", for anyone aiming at it or planning around it. It short-circuits to the
current position when the path is completed, empty or already consumed, and otherwise runs
the algorithm above on a *copy* of the travel-point index so the query has no side effects.

## `speed`

**Contract** — reports the creature's speed for external consumers. Zero when the achieved
speed is zero. Otherwise, if the physics character is enabled, the horizontal component of
the character's actual velocity projected onto its heading; if not, the filtered achieved
speed.

**Notes** — the two answers differ, and deliberately: when the solver owns the creature the
truth is what the solver did, and when it does not the truth is what the mover intended.
Asking the disabled character would return the stale value from whenever it was last
enabled.

## `apply_collision_hit`

**Contract** — if the creature is alive and the physics character accumulated a non-zero
health loss from contacts this step, convert that into a hit on the creature: the
accumulated amount as damage, the direction, position, hit type and blaming entity taken
from the character's collision damage record, and the contacted bone as the hit bone. Side
effect only; returns nothing.

**Invariants** — this must be called on both branches where the physics solver moved the
creature, and on the standing-still branch too. Skipping it on the standing-still branch
would make a creature immune to being run over while idle, which is a bug players notice
with vehicles.

**Notes** — the damage source is read out of the physics layer rather than recorded by the
mover, because the contact happened inside the solver step and only the solver knows who
was on the other side of it. This is the seam where rigid-body contacts re-enter the game's
damage model.
