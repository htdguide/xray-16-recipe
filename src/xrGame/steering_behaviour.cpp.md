# src/xrGame/steering_behaviour.cpp

> Six forces that pull a flying or driving thing around, and the accumulator that adds them
> up into one acceleration per frame.

**Needs** — [`steering_behaviour.h`](steering_behaviour.h.md)
**Used by** — [`steering_behaviour.h`](steering_behaviour.h.md)
**Tier floor** — T2: vector arithmetic per behaviour per frame.

## Purpose

Not every moving thing in the game paths. Helicopters, and anything else that moves
continuously through open space rather than along a navigation mesh, are steered instead:
each frame, a set of independent behaviours each produce an acceleration vector, the vectors
are summed, and the result is handed to the motion integrator. There is no plan and no
route — the trajectory is whatever the sum of forces produces.

The distinction from the pathfinding creatures is worth stating once: a pathed creature
decides *where to go* and then walks there; a steered object is *pushed*. That makes steered
motion smooth and unpredictable and makes it impossible to guarantee arrival, which is why
the two systems coexist rather than one replacing the other.

## The common shape

Every behaviour is the same three things: a **supplier** that refreshes the behaviour's
inputs from the world each frame and reports whether the behaviour is still meaningful; an
**enabled** flag; and a function producing this frame's acceleration. A behaviour that has
no opinion returns zero rather than declining, so the accumulator never has to special-case
anything.

```text
RECORD BehaviourParams                     # the supplier, subclassed per behaviour
  enabled         : bool
  factor          : vector                 # the three coefficients below
  min_factor_dist : real
  update()        -> bool                  # refresh inputs; false means "retire me"
```

### The distance falloff

One formula, shared by five of the six behaviours, and the most load-bearing thing in the
file:

```text
FUNCTION dist_factor(factor, distance) -> real
  r := min(distance, min_factor_dist)      # NOTE: min, not max — see below
  RETURN factor.x + factor.y / r + factor.z / (r * r)
```

The three components of a behaviour's factor are a **constant** pull, an **inverse-distance**
pull and an **inverse-square** pull, and an author tunes a behaviour by choosing how much of
each. Constant-only gives a force that does not care how far away the thing is; inverse-square
gives one that is negligible at range and violent up close, which is how a separation force
is made to feel like a repulsion rather than a leash.

`min_factor_dist` is supposed to *clamp the falloff near zero*, so that a distance
approaching zero does not produce an unbounded force. As written the clamp is applied the
wrong way round: the helper takes the **smaller** of the distance and the limit, so the force
saturates at the limit for every distance beyond it and continues to grow without bound as
the distance approaches zero — exactly inverted from the stated intent. The file's own local
`max` helper is also written as a minimum, which is the likely origin. A rebuild must decide
deliberately: the correct clamp is `max(distance, min_factor_dist)`, and switching to it
changes the tuned feel of every behaviour that uses a non-zero inverse term.

## `evade`

**Contract** — push *toward* a destination, but only while within a maximum range; beyond
it, nothing.

```text
FUNCTION acceleration() -> vector
  to_dest := dest - pos
  d       := max(length(to_dest), tiny)
  IF d > max_evade_range
    RETURN zero                            # out of range: this behaviour has no opinion
  direction := (length(to_dest) > tiny) ? normalize(to_dest) : random_unit_vector()
  RETURN direction * dist_factor(d)
```

**Notes** — the random direction when the object is exactly on its destination is not a
guard against division by zero; it is a *decision*. Something told to evade from where it
already stands must go somewhere, and any direction is as good as another. Returning zero
instead would leave it pinned.

## `pursue`

**Contract** — close on a destination, with an arrival band and a braking band. Three
regimes:

```text
FUNCTION acceleration() -> vector
  to_dest := dest - pos
  d       := length(to_dest)
  IF d < arrive_range
    RETURN zero                            # arrived
  direction := to_dest / d
  IF d <= change_vel_range
    # braking: solve for the acceleration that reaches arrive_vel over the remaining distance
    sum := vel + arrive_vel
    IF sum < tiny
      RETURN zero
    time_to_go := 2 * d / sum              # mean-speed estimate of the remaining time
    RETURN direction * ((arrive_vel - vel) / time_to_go)
  RETURN direction * dist_factor(d)        # far: just pull
```

**Invariants** — the middle regime is the interesting one, and it is not a falloff: it is a
solved acceleration. Given the current speed and the speed wanted on arrival, the time to
cover the remaining distance is estimated from the mean of the two, and the acceleration is
the speed change divided by that time. Consequences: approaching too fast produces a
*negative* acceleration — real braking — and the estimate is exact for constant acceleration,
so an object that arrives in a straight line arrives at very close to its intended speed.

`arrive_range` and `change_vel_range` are the two radii around the target: outside the
larger one the object is simply pulled in; inside it, it brakes; inside the smaller one it
is done. A rebuild must keep them distinct or the object either overshoots or stops short.

The zero-sum-velocity case — stationary object wanting to arrive stationary — returns
nothing, and the original flags it as unresolved. It is genuinely ill-posed: the formula
cannot say how to travel a distance at zero speed. Some other behaviour has to provide the
initial push.

## `restrictor`

**Contract** — a leash. Zero while within an allowed range of an anchor point; beyond it, a
pull back toward the anchor scaled by the falloff.

**Notes** — the simplest behaviour in the file, and the one that makes the others safe:
whatever the rest produce, something bounded by a restrictor cannot wander out of its
authored area. The word is the project's own — a volume that constrains where an entity may
go — used here as a point and a radius rather than as a navigation-mesh region.

## `wander`

**Contract** — a smoothly varying push that keeps something moving without a destination.
Operates in one of three chosen coordinate planes.

```text
FUNCTION acceleration() -> vector
  wander_angle := wander_angle + (coin_flip ? +1 : -1) * angle_change   # a random walk
  planar := project current direction onto the chosen plane
  IF length(planar) < tiny
    RETURN zero
  rotated := planar rotated by wander_angle within the plane
  RETURN normalize(normalize(rotated) + direction * conservativeness) * factor.x
```

**Invariants** — the angle is a **random walk, not a random value**: each frame it steps by
a fixed amount in one of two directions and is never reset. That is what makes the motion
look like drifting rather than like twitching, and it is why the behaviour has to keep state
between frames at all.

`conservativeness` blends the rotated direction back toward the current heading. High values
produce a lazy arc; zero produces something that turns as fast as the angle allows. Only the
constant component of the factor is used — a wander has no distance to fall off with.

The plane choice exists because a wandering helicopter should not wander in altitude the way
it wanders in heading.

## `containment`

**Contract** — feel ahead with a set of probes and push away from whatever they hit. The only
behaviour that queries geometry.

```text
FUNCTION acceleration() -> vector
  build an orthonormal frame from the object's heading and up vector
  steer := zero
  FOR EACH probe IN probes                 # probes are authored in the object's own frame
    world_probe := probe expressed in the world frame
    IF test_obstacle(world_probe) HITS at point with surface normal
      d := distance(point, pos)
      f := dist_factor(d)
      thrust := heading * (-f)             # slow down
      turn   := right * dot(normalize(normal), right) * turn_factor * f
      steer  := steer + thrust + turn
  RETURN steer
```

**Invariants** — each hit contributes two things and the split is the behaviour: a **thrust**
straight back along the heading, which slows the object down, and a **turn** sideways whose
sign and size come from how much the obstacle's surface normal points to one side. An
obstacle dead ahead with a face-on normal contributes braking and almost no turn; one glanced
at an angle contributes mostly turn. That is what produces a plausible swerve instead of a
stop.

The probes are authored in the object's own frame and re-expressed in the world each frame,
so a rotating object sweeps its probes with it. Their layout is the object's field of
attention and is entirely tuning.

`turn_factor` scales turning against braking — the object's willingness to go around rather
than slow down.

## `grouping`

**Contract** — the flocking behaviour: cohesion toward the centre of the neighbours, plus
separation from each neighbour that is too close. One pass over the neighbours produces
both.

```text
FUNCTION acceleration() -> vector
  steer := zero;  sum := zero;  count := 0
  FOR EACH neighbour IN neighbours                # supplied by the params object
    away := pos - neighbour
    d    := length(away)
    IF d < max_separate_range
      direction := (d > tiny) ? away / d : random_unit_vector()
      steer := steer + direction * dist_factor(separation_factor, d)
    sum := sum + neighbour;  count := count + 1
  IF count == 0
    RETURN zero
  centre := sum / count
  to_centre := centre - pos
  IF length(to_centre) > tiny
    steer := steer + normalize(to_centre) * dist_factor(cohesion_factor, length(to_centre))
  RETURN steer
```

**Invariants** — the two halves have *separate factor triples*, and that is what makes
flocking work: cohesion is tuned as a weak long-range constant pull and separation as a
strong short-range inverse-square push, so neighbours gather but do not collide. Giving them
one factor collapses the behaviour.

Separation is bounded by its own range and cohesion is not — a neighbour too far to repel
still counts toward the centre. Every neighbour contributes a separate separation term but
only one shared cohesion term, so separation scales with crowding while cohesion does not.

Neighbours are delivered by an explicit start/next/done walk on the params object rather
than as a collection, so the caller can stream them out of a spatial query without
materialising a list.

## `manager`

**Contract** — one frame: refresh every behaviour, drop the ones that retired, sum the
accelerations of the ones that are enabled.

```text
FUNCTION acceleration() -> vector
  remove_scheduled()                       # retire what last frame marked
  total := zero
  FOR EACH behaviour IN behaviours
    IF NOT behaviour.params.update()
      schedule_remove(behaviour)           # marked now, removed at the start of next frame
    IF behaviour.params.enabled
      total := total + behaviour.acceleration()
  RETURN total
```

**Invariants** — three decisions.

*Removal is deferred by one frame.* A behaviour that retires during the walk is marked and
removed at the start of the next one, so the collection is never modified while being walked.

*A behaviour that retires this frame still contributes this frame.* The enabled test comes
after the refresh and does not consult the retirement, so a behaviour's last acceleration is
applied. Whether that was intended is not recoverable; it is at most one frame of a force
that was about to disappear.

*The sum is unweighted.* All relative strength lives in the behaviours' own factors, so a
rebuild must not introduce per-behaviour weights at the accumulator — that would double the
tuning surface and silently rescale every authored configuration.

The manager owns its behaviours and destroys them on teardown.

## What could not be recovered

- Whether the inverted distance clamp (and the local minimum-named-maximum helper behind it)
  is a bug or was compensated for during tuning. The behaviour is reproducible; the intent
  is not.
- Whether a behaviour retiring mid-frame was meant to contribute that frame's acceleration.
- `pursue`'s zero-combined-velocity case, which the original itself marks as unresolved.
