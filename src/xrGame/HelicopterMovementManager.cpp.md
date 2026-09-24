# src/xrGame/HelicopterMovementManager.cpp

> Where the helicopter is going: path following over authored patrol paths, a procedurally built orbit, the arrival test, terrain-following altitude, and the speed-dependent turn rates the flight model asks for.

**Needs** — [`helicopter.h`](helicopter.h.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`Level.h`](Level.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path_storage.h`](../xrAICore/Navigation/PatrolPath/patrol_path_storage.h.md) · [`game_object_space.h`](game_object_space.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`Helicopter.cpp`](Helicopter.cpp.md)
**Tier floor** — T2: graph walking, a ray query and interpolation; no device or layout concern

## Purpose

The helicopter's destination, and nothing else. The flight model in
[`Helicopter.cpp`](Helicopter.cpp.md) asks this record three questions every step — where
am I going, how fast may I turn at this speed, and have I arrived — and this file answers
all three.

Three ways of having a destination exist and they are deliberately different kinds of
thing:

- **a point**, set by script, reached once and then forgotten;
- **an authored patrol path**, a named graph in the level data, walked vertex by vertex
  and looping by whatever edges the author gave it;
- **an orbit**, which does not exist in the level data at all — it is a patrol path built
  at runtime, in a circle, and thrown away when the helicopter is told to do something
  else.

That the orbit is *synthesized as a patrol path* rather than being its own mode is the
load-bearing decision: it means one path follower serves all three, and the orbit
inherits arrival behaviour, saving and the on-point callback for free.

## State

See [`helicopter.h`](helicopter.h.md) for the record; the parts this file owns:

```text
RECORD MovementState
  kind             : ENUM { none, to_point, patrol_path, round_path, landing, take_off }
  path             : optional<PatrolPath>   # authored, or synthesized for an orbit
  vertex           : optional<PathVertex>   # where on that path we are heading
  path_is_ours     : bool    # true only for a synthesized orbit; governs destruction
  patrol_name      : text
  patrol_begin_idx : int
  desired          : vec3    # the point currently being flown to
  current          : vec3    # the integrator's position, shadowed here
  path_heading, path_pitch : real   # the flight direction, not the airframe attitude
  speed, acceleration : real
  max_speed        : real
  accel_forward, accel_brake : real
  arrival_speed    : real    # the speed to be doing on arrival
  arrival_radius   : real    # how close counts as arrived
  min_altitude, safe_altitude_add : real
  orbit_centre, orbit_radius : ...   # remembered so an orbit can be rebuilt on load
  orbit_reversed   : bool
```

Invariants:

- The path is owned by this record **only** when it was synthesized. An authored path
  belongs to the level's path storage and must never be destroyed here. Every path swap
  and the teardown consult the ownership flag.
- An orbit's entry vertex is marked with a flag, and that flag means "slow to a stop
  here" — the arrival speed is reported as zero at a flagged vertex. It is cleared once
  passed, so the helicopter stops only on the way *in*.
- The saved orbit is not saved as a path; it is saved as its centre, radius and direction
  and rebuilt on load.

## `Load`

**Contract** — reads the two angular-rate endpoints for pitch and heading (the rate at a
standstill and the rate at maximum speed), the two linear acceleration rates, the arrival
radius, the maximum speed, the minimum altitude, the safe-altitude margin, and a flag
selecting an alternative heading-rate curve. Derives the linear coefficients of the
rate-versus-speed lines from the endpoints.

**Invariants** — the turn rates are expressed in configuration as *two samples* — what
the rate is at zero speed and what it is at maximum speed — and the slope is derived.
That is the right shape for a tuner and a rebuild should keep it.

## `GetAngSpeedPitch` / `GetAngSpeedHeading`

**Contract** — the maximum angular rate at a given speed. Pitch is always the linear
interpolation between the two configured samples. Heading is the same by default, or, when
the alternative flag is set, a hyperbolic falloff instead.

**Notes** — the alternative heading curve is a later addition with a different shape:
rather than a line, the rate falls off as the reciprocal of a linear function of speed, so
the helicopter keeps more of its turn authority at low speed and loses it faster at high
speed. The source keeps both and switches by a configuration flag; three further
candidate formulas survive as comments. A rebuild should pick one and say so.

## `Update`

**Contract** — the once-a-frame path-follower tick. Dispatches on the movement kind: a
point destination and a path destination each get their own arrival check; the landing
and take-off kinds do nothing at all.

**Notes** — the landing and take-off kinds are declared and reachable through the saved
state but have no implementation. They are unfinished features; a rebuild should not
carry them.

## `AlreadyOnPoint`

**Contract** — the arrival test. True when within a tenth of a metre, and also true when
inside the arrival radius **and moving away** — that is, when one more step would increase
the distance.

**Invariants** — the "moving away" test is what makes a fast helicopter count as arrived
at the moment of closest approach rather than turning back to touch the point exactly.
Without it a gunship would circle its waypoints. The test projects one fixed step forward
along the *flight heading at zero pitch* and compares distances.

```text
FUNCTION already_on_point() -> bool
  distance = |desired - current|
  IF distance <= 0.1 THEN RETURN true
  IF distance < arrival_radius THEN
    ahead = current advanced one fixed step along the flight heading, level
    RETURN |desired - ahead| > distance        # closest approach is now or past
  RETURN false
```

## `UpdatePatrolPath`

**Contract** — on arrival at a path vertex, fire the on-point script callback with the
distance, the position and the vertex identifier, clear the orbit entry flag if this was
the flagged vertex, and advance to the first vertex the current one has an edge to. A
vertex with no outgoing edges ends the movement.

**Invariants** — the callback fires *before* the advance, so a script that redirects the
helicopter from inside the callback wins. The successor is simply the first edge, so a
branching patrol path is walked deterministically but arbitrarily; authors are expected to
supply a single cycle.

## `UpdateMovToPoint`

**Contract** — on arrival at a script-set point, fire the on-point callback with a vertex
identifier of "none" and stop. The helicopter then hovers where it is, braking to a halt.

## `SetDestPosition`

**Contract** — fly to a point. Sets the destination, switches the kind, and destroys any
synthesized orbit path that was in use.

## `goPatrolByPatrolPath`

**Contract** — follow a named authored path from a starting vertex. Releases any
synthesized path first, looks the path up in the level's path storage, takes the starting
vertex's position as the destination, and marks the path as not owned.

## `goByRoundPath`

**Contract** — orbit a centre at a radius, in a given direction. Builds a closed patrol
path of points around the circle, picks an entry vertex, marks it, and starts following.

**Invariants** — three decisions here are load-bearing:

- **The orbit is refused if it is too tight to fly.** The minimum radius is the product of
  the maximum speed and the heading rate at that speed; a smaller circle cannot be tracked
  and the request is reported and dropped rather than producing a helicopter that spirals.
- **The entry vertex must be further away than the stopping distance.** The nearest vertex
  the machine could *not* stop before is chosen, so it always enters the circle with room
  to decelerate. The stopping distance is computed from the current speed and the braking
  rate.
- **Calling it again while already orbiting reverses the direction.** That is how a
  script makes a gunship switch from clockwise to counter-clockwise: it asks for the same
  orbit again.

```text
FUNCTION go_by_round_path(centre, radius, clockwise)
  IF already orbiting THEN clockwise = NOT clockwise        # a repeat request reverses
  min_radius = max_speed * heading_rate_at(max_speed)
  IF radius < min_radius THEN report and RETURN             # untrackable circle

  release any synthesized path
  points = circle of points around centre at radius, one every 30 metres of arc,
           all at the centre's altitude
  IF clockwise THEN reverse all but the first                # keeps the entry point fixed
  build a patrol path: a vertex per point, an edge from each to the next, and one
    edge closing the last back to the first
  stopping_distance = the distance needed to brake to a halt from the current speed
  entry = the nearest vertex whose distance exceeds stopping_distance
  mark entry with the "stop here" flag
  destination = entry's position ; kind = orbit
```

**Notes** — the spacing of thirty metres between orbit points is a file-scope constant
with no derivation. It sets how polygonal the orbit is; too coarse and the helicopter
visibly flies a polygon, too fine and every vertex triggers an arrival callback.

The synthesized path's vertices are constructed with null navigation-graph references,
because a helicopter does not use the navigation mesh at all. The patrol-path type demands
them; a rebuild with a plainer waypoint list does not.

## `SetPointFlags`

**Contract** — change a vertex's flags by constructing a replacement vertex and assigning
it over the original. Used only to set and clear the orbit's entry mark.

**Notes** — the replacement is allocated and the original is *not* released; the original's
deletion is commented out in the source. This leaks one small record per orbit entry and
exit. A rebuild with a mutable flag field avoids the whole mechanism.

## `getPathAltitude`

**Contract** — place a point at a given height above whatever is beneath it. Casts a ray
straight down from the top of the level's bounding volume, sets the point's altitude to
the surface it finds plus the requested margin, and clamps the result into the level's
vertical extent.

**Invariants** — the ray starts from the top of the level rather than from the point, so
the result does not depend on where the point currently is vertically — a point below
terrain still lands on top of it. The clamp's upper bound is the level ceiling plus the
margin, which is what stops a helicopter from being sent above the skybox.

## `GetSafeAltitude`

**Contract** — the level's ceiling plus a configured margin: the altitude at which nothing
in the level can be hit. Scripts use it to send a gunship away.

## `GetSpeedInDestPoint` / `SetSpeedInDestPoint`

**Contract** — the speed to be doing on arrival. Reports **zero** — a full stop — when the
destination is the flagged entry vertex of a synthesized orbit, and the configured value
otherwise.

**Invariants** — this override is the only consumer of the entry flag, and it is why a
helicopter entering an orbit slows and then picks up speed around the circle.

## `save` / `load`

**Contract** — persists the movement kind, the patrol path's name and starting index, the
speed and acceleration limits, the arrival speed and radius, the destination, the live
speed, acceleration, position and flight angles, and the orbit's centre, radius and
direction. A path-following helicopter additionally writes the identifier of the vertex it
is heading for.

**Invariants** — on load an authored path is re-looked-up by name and the vertex by
identifier; an orbit is *rebuilt from scratch* from its centre, radius and direction,
with the direction inverted on the call because rebuilding while already orbiting would
otherwise reverse it. That double negative is a consequence of the reverse-on-repeat rule
and is easy to get wrong.

## `net_Destroy`

**Contract** — release the synthesized path, and only that. An authored path belongs to
the level.

## `GetCurrVelocityVec`

**Contract** — the unit flight direction, reconstructed from the stored heading and pitch
rather than from any difference of positions. Scripts use it to lead a target.

## `OnRender`

**Contract** — a debug hook whose body is entirely commented out. It once drew the patrol
path's vertices and the aim lines. Nothing remains; a rebuild should omit it.
