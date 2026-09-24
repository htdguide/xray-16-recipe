# src/xrParticles/particle_actions_collection.cpp

> The action catalogue itself: for each of the thirty-one verbs, the parameters it carries and
> exactly what it does to every live particle in one step.

**Needs** — [`particle_actions_collection.h`](particle_actions_collection.h.md) · [`particle_effect.h`](particle_effect.h.md) · [`particle_core.h`](particle_core.h.md) · [`noise.h`](noise.h.md) · [`psystem.h`](psystem.h.md)
**Used by** — reached through its declarations in [`particle_actions_collection.h`](particle_actions_collection.h.md); callers name that, not this file.
**Tier floor** — T1: the inner loops run over hundreds of particles thirty-one times a step
inside a 60 Hz budget, and one of them hand-vectorizes a four-wide float operation. The
*decisions* are all T2; the throughput is not.

## Purpose

This is the chapter. Everything else — pools, domains, handles, file formats — exists to let
these run. An effect is a sequence of them; the look of every fire, spark, mist and muzzle
flash in the game is the choice of which ones, in what order, with what parameters.

The file is one unit because the actions share a small vocabulary and nothing else: they do
not call each other, they hold no shared state, and each one is independently replaceable. A
rebuild may split it per action with no loss.

Every action's loop is a candidate for data parallelism across the pool and **none of them
take it**. The file reaches for the project's parallel-loop facility and then uses it nowhere;
the one action expensive enough to justify it carries an explicit instruction not to. The
reason is the same in every case: a few hundred particles times a few dozen float operations
is below the cost of distributing the work, and several actions mutate the pool's size, which
a parallel loop cannot do. A rebuild should keep the actions serial and find its parallelism
across *effects*, of which a level has hundreds.

## State

None of its own. Each action is a record of authored parameters, listed under its heading
below, and every action's parameters obey the same two rules:

- **Local and world copies.** A parameter with a position, direction or domain exists twice.
  The local copy is authored and immutable; the world copy is derived by the action's
  `transform` whenever the emitter moves. Simulation reads only the world copy. Below, only
  the world copy is named.
- **Sentinels, not options.** `max_radius` and `look_ahead` at or above the unbounded sentinel
  (`1e16`, see [`psystem.h`](psystem.h.md)) select a branch with no range test at all rather
  than a very large one. Two copies of each loop exist in the source for this reason; that
  duplication is incidental, the behaviour is not.

Three conventions recur and are stated once here rather than in every entry:

- **`epsilon` is a softening term**, always added to a squared distance in a denominator, so
  that a particle passing exactly through the singularity gets a large but finite force
  instead of infinity. It is not a tolerance and never compared against.
- **`magnitude` is multiplied by the step duration** before use, so every force action is
  parameterized per second.
- **Kill loops run backwards** over the pool, because removal moves the last live particle into
  the hole ([`particle_effect.cpp`](particle_effect.cpp.md)).

## `Move`

**Contract** — integrates the pool. Parameters: none.

```text
FUNCTION execute(pool, dt)
  FOR EACH particle p IN pool
    p.age  = p.age + dt
    p.posB = p.pos              # the previous position, for motion blur and collision
    p.pos  = p.pos + p.vel · dt
```

**Invariants** — this is the *only* action that advances position from velocity and the only
one that advances age. An effect without it has particles that never move and never die of old
age. Its position in the list decides whether the force actions before it apply to this step's
motion or the next one's; shipped effects put it last.

**Notes** — it overwrites the second position slot, which is also where the restore action
keeps its target and where the source action can latch a birth position. Move and restore in
the same list are mutually destructive; move and vertex-B-tracking emission are not, because
the source writes the slot at birth and move immediately starts using it as a trail.

## `Source`

**Contract** — the only action that creates particles. Parameters: six domains (`position`,
`velocity`, `rot`, `size`, `color`), an `alpha`, a `particle_rate` in particles per second, an
initial `age` with a standard deviation `age_sigma`, an inherited velocity with an
`inheritance_factor`, and three flags — silent, single-size, and vertex-B-tracking. Emits
nothing while silent. Consumes several random draws per emitted particle. Never exceeds the
pool's capacity.

```text
FUNCTION execute(pool, dt)
  IF silent THEN RETURN

  # Fractional rates must not round to zero or the effect's density depends on
  # the step length. Emit the whole part and the fraction as a probability.
  n = floor(particle_rate · dt)
  IF uniform_random() < particle_rate · dt - n THEN n = n + 1
  n = min(n, pool.capacity - pool.live)

  REPEAT n TIMES
    pos   = position.generate()
    size  = size.generate()
    IF single_size THEN size = (size.x, size.x, size.x)   # one authored axis drives all three
    rot   = rot.generate()
    vel   = velocity.generate() + inherited_velocity
    col   = color.generate()                              # components are 0..1 RGB
    birth_age = age + normal_random(age_sigma)
    second = vertex_b_tracks ? pos : UNSPECIFIED
    pool.add(pos, second, size, rot, vel, pack_color(alpha, col), birth_age)
```

**Invariants** — the emission domains are sampled in a fixed order: position, size, rotation,
velocity, colour, then age. With a single shared random source, that order is part of the
observable stream; a rebuild that samples in a different order produces different particles
from the same file even with the same generator.

The inherited velocity is not authored directly. It is recomputed on every emitter move as the
emitter's own velocity times the authored inheritance factor
([`particle_manager.cpp`](particle_manager.cpp.md)), which is how sparks from a moving vehicle
trail behind it.

**Notes** — the dithering of the fractional particle is what lets an effect authored at, say,
7 particles per second work at a 33 ms step, where the exact count per step is 0.23. Without
it every such effect emits nothing.

When vertex-B tracking is off, the second position slot is filled from a value that was never
assigned — the birth position is whatever the previous occupant of that pool entry left, or
uninitialized memory on the first use. This is a real defect and it is invisible in practice
because every shipped effect either sets the tracking flag or runs a move action, which
overwrites the slot on the particle's first step. A rebuild should initialize it to the birth
position and accept the divergence for the one frame it could matter.

The initial age is drawn from a *normal* distribution, so it can be negative. A negative age
makes a particle immune to the kill-old action for the first fraction of a second and puts it
outside every colour action's time window until it catches up.

## `KillOld`

**Contract** — retires particles by age, and publishes the effect's lifetime to the actions
after it. Parameters: `age_limit`, and `kill_less_than` choosing which side of the threshold
dies. Fires the pool's death callback for each retirement. Walks the pool backwards.

```text
FUNCTION execute(pool, dt, lifetime_hint)
  lifetime_hint = age_limit            # every later action's notion of "a lifetime"
  FOR i FROM pool.live - 1 DOWN TO 0
    young = pool[i].age < age_limit
    IF young = kill_less_than THEN pool.remove(i)
```

**Invariants** — the published lifetime is read by the colour action and by nothing else. An
effect whose colour action precedes its kill-old action, or that has no kill-old action, gets
the default lifetime of one second. See
[`particle_manager.cpp`](particle_manager.cpp.md).

**Notes** — the inverted form (`kill_less_than` true) retires the *young* and keeps the old. It
exists for effects that hand particles off between two systems.

## `Gravity`

**Contract** — constant acceleration. Parameter: a `direction` vector that is itself the
acceleration per second.

```text
FOR EACH particle p : p.vel = p.vel + direction · dt
```

## `Damping`

**Contract** — multiplicative decay of velocity toward zero, per axis, applied only to
particles whose speed falls inside a band. Parameters: a per-axis `damping` factor, and
`vlowSqr`/`vhighSqr` bounding the band as squared speeds.

```text
FUNCTION execute(pool, dt)
  # Expressed so the factor means "per second" regardless of the step length:
  # a damping of 0.9 removes a tenth of the speed per second, not per step.
  scale = 1 - (1 - damping) · dt
  FOR EACH particle p
    s2 = |p.vel|²
    IF s2 >= vlowSqr AND s2 <= vhighSqr THEN p.vel = p.vel componentwise-times scale
```

**Notes** — the band makes damping selective: authors use it to settle slow particles while
leaving fast ones alone, or the reverse. The linearization of the decay is exact only for
small steps; at the chapter's 33 ms step and factors near 1 the error is invisible, and at a
damping factor above 1 the formula amplifies, which authors use deliberately.

## `SpeedLimit`

**Contract** — clamps speed into `[min_speed, max_speed]`, preserving direction. A particle at
exactly zero velocity is left alone rather than being given an arbitrary direction.

```text
FOR EACH particle p
  s2 = |p.vel|²
  IF s2 < min_speed² AND s2 != 0 THEN p.vel = p.vel · (min_speed / sqrt(s2))
  ELSE IF s2 > max_speed²        THEN p.vel = p.vel · (max_speed / sqrt(s2))
```

## `TargetColor`

**Contract** — eases colour and alpha toward a target, but only while a particle's age lies in
a window expressed as fractions of the effect's lifetime. Parameters: a `color` as three 0..1
components, a target `alpha`, a `scale` giving the fraction of the remaining difference closed
per second, and `time_from`/`time_to` defaulting to 0 and 1.

```text
FUNCTION execute(pool, dt, lifetime_hint)
  k = scale · dt
  FOR EACH particle p
    IF p.age < time_from · lifetime_hint OR p.age > time_to · lifetime_hint THEN CONTINUE
    unpack p.color into r,g,b,a
    move each of r,g,b toward the target component by k of the difference
    move a toward the target alpha the same way
    repack
```

**Invariants** — the window is relative, so one authored colour ramp works for effects of
different lifetimes — provided the kill-old action ran first to publish the lifetime.

**Notes** — the easing is exponential, not linear: each step closes a fraction of what remains,
so the target is approached and never exactly reached. Authors compensate by overshooting the
target colour.

Shadow-of-Chernobyl-era files do not carry the two window fields at all; the loader stops
early and the defaults 0 and 1 stand, which is the whole-lifetime window the older engine
always used. See
[`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md).

## `TargetSize`

**Contract** — the same easing applied to size, with an independent rate per axis. Parameters:
a target `size` and a per-axis `scale`. No time window.

```text
FOR EACH particle p : p.size = p.size + (size - p.size) componentwise-times (scale · dt)
```

## `TargetVelocity`

**Contract** — eases velocity toward a target vector at rate `scale` per second. The target is
a world-space direction re-derived from the authored one on every emitter move, so a wind
authored in an emitter's local frame turns with it only if that action opts into rotation.

```text
FOR EACH particle p : p.vel = p.vel + (velocity - p.vel) · (scale · dt)
```

## `TargetRotate`

**Contract** — eases the *magnitude* of a particle's spin toward a target while preserving its
direction. Parameters: a target rotation (authored as three components, of which only the
first is used) and a `scale`.

```text
FUNCTION execute(pool, dt)
  target = |rot.x| ; k = scale · dt
  FOR EACH particle p
    # The sign of the current spin is the direction the particle is turning and must
    # survive: only the amount changes.
    signed_k = p.rot >= 0 ? k : -k
    p.rot = p.rot + (target - |p.rot|) · signed_k
```

**Notes** — a particle whose spin starts at exactly zero eases toward the positive target,
because zero counts as non-negative. The catalogue's separate "derivative" kind code for this
action resolves to this same implementation; see [`psystem.h`](psystem.h.md).

## `CopyVertexB`

**Contract** — latches each particle's current position into its second slot when the
`copy_pos` flag is set; otherwise does nothing. Parameters: that one flag.

**Notes** — the slot's meaning is whatever the list makes it. Placed before a restore action,
this pins the restore target to where the particles are now; placed anywhere near a move
action, it is immediately overwritten.

## `Explosion`

**Contract** — a Gaussian shock shell expanding from a centre, pushing particles outward as it
passes them. Parameters: a `center`, the shell's `velocity`, a force `magnitude` at unit
radius, a `stdev` setting the shell's thickness, an `age` counting the time since detonation,
and an `epsilon`. Carries time: `age` advances every step and is reset to zero when the effect
is played.

```text
FUNCTION execute(pool, dt)
  radius = velocity · age                   # where the shell is now
  FOR EACH particle p
    dir  = p.pos - center ; dist = |dir|
    # Force peaks on the shell and falls off as a gaussian on both sides of it.
    shell = exp(-0.5 · ((radius - dist)/stdev)²) / (stdev · sqrt(2·pi))
    p.vel = p.vel + dir · (shell · magnitude · dt / ((dist + tiny) · (dist² + epsilon)))
  age = age + dt
```

**Notes** — the force is radial and falls off as the inverse cube of distance overall (one
factor of distance normalizes `dir`, two more soften the force), on top of the shell's
Gaussian. A single-shot explosion is authored by playing the effect; without a play, `age`
keeps growing and the shell passes out of the world and never returns.

## `Turbulence`

**Contract** — bends velocity along the gradient of a fractal noise field without changing its
magnitude. Parameters: `frequency`, `octaves`, `magnitude`, `epsilon` used here as a
finite-difference step, an `offset` direction, and an `age`. Carries time: `age` advances every
step and resets when the effect is played. Initializes the noise tables on first use. Does not
follow the emitter — its transform is empty, so the field is world-fixed and particles swim
through it.

```text
FUNCTION execute(pool, dt)
  ensure noise tables built
  age = age + dt
  FOR EACH particle p
    # The sample point drifts with age: the field scrolls past the particles,
    # which is what makes smoke curl rather than settle into a static pattern.
    q = p.pos + offset · age
    d = fractalsum3(q, frequency, octaves)
    # Forward differences along each axis give the field's gradient. epsilon is the
    # step, so it trades noise in the gradient against how local it is.
    g = ( fractalsum3(q + (epsilon,0,0), ...) - d,
          fractalsum3(q + (0,epsilon,0), ...) - d,
          fractalsum3(q + (0,0,epsilon), ...) - d ) · magnitude
    speed_before = |p.vel|
    p.vel = (p.vel + g)
    p.vel = p.vel · (speed_before / |p.vel|)      # steer only; never speed up or slow down
```

**Invariants** — speed is preserved exactly. That is the action's defining property: turbulence
is a direction field, and any speed change is the job of damping or a speed limit.

**Notes** — four fractal-sum evaluations per particle per step makes this the most expensive
action in the catalogue by an order of magnitude. The source hand-vectorizes the renormalize
with four-wide float operations and carries an explicit instruction not to parallelize the
loop across threads: measurement said the single-threaded version won, because the work per
particle is small relative to the cost of distributing it. A rebuild should express the inner
block as four-wide float operations and leave the loop serial.

Renormalizing divides by the new speed. A particle at exactly zero velocity produces a
division by zero here, which no shipped effect reaches because turbulence is always paired with
an emitter that gives particles a non-zero initial velocity.

## `Avoid`

**Contract** — steers particles away from a domain *before* they reach it, preserving speed.
Parameters: a `position` domain, a `look_ahead` in time units, a `magnitude` setting how
sharply to turn, and an `epsilon`. Supports five domain kinds — plane, rectangle, triangle,
disc and sphere — and silently does nothing for the other six.

The steering itself is one formula shared by every domain kind; the kinds differ only in how
they compute the escape direction and the time to impact.

```text
FUNCTION steer(p, escape_direction, time_to_impact, magnitude·dt, epsilon)
  speed = |p.vel| ; heading = p.vel / speed
  # Blend the escape direction into the heading, weighted by how soon the impact is,
  # then restore the original speed: avoidance turns, it never brakes or accelerates.
  blended = escape_direction · (magnitude·dt / (time_to_impact² + epsilon)) + heading
  p.vel   = blended · (speed / |blended|)
```

Per kind:

```text
Plane      : dist = signed distance of p.pos from the plane
             IF look_ahead is bounded AND dist >= look_ahead THEN skip
             steer(p, plane normal, dist, ...)      # always pushed to the positive side

Rectangle,
Triangle   : predict = p.pos + p.vel · dt · look_ahead
             skip unless p.pos and predict lie on opposite sides of the plane
             t    = time at which the segment crosses the plane
             hit  = p.pos + p.vel · t
             express hit - origin in the domain's (u,v) coordinates via the
               precomputed inverse of the plane basis; skip if outside the face
             escape = the shortest vector from the hit point to one of the face's edges
             steer(p, normalize(escape), t, ...)

Disc       : as above, but the in-face test is the annulus test on the hit radius,
             and the escape direction is radially outward from the disc centre.
             Note the disc keeps its plane offset in a different field than the
             other planar kinds; see particle_core.cpp.

Sphere     : ray-sphere intersection along the heading; skip if the ray misses, if
             the hit is behind, or if it is further than speed · look_ahead
             escape = the component of the heading turned away from the centre,
                      built as heading crossed with (heading crossed with centre-offset)
             steer(p, escape, distance to the hit, ...)
```

**Invariants** — speed is preserved. Avoidance never removes a particle and never moves one; it
only turns velocities, so a particle that is already inside the domain is not pushed out.

**Notes** — `look_ahead` is in time units for the planar kinds (the prediction multiplies it by
the step) but the plane kind compares it against a *distance*, and the sphere kind compares
speed times look-ahead against a distance. The three readings are inconsistent and frozen;
authors tuned against whichever kind they used.

The rectangle's edge search computes its third and fourth candidate edges with the same basis
vector, so the fourth is a duplicate of the third and one edge of the rectangle is never the
chosen escape direction. Particles approaching that edge are steered toward a neighbouring one.
A rebuild that fixes this changes the behaviour of every rectangle-avoid in the shipped data.

## `Bounce`

**Contract** — reflects velocity off a domain's surface with restitution and tangential
friction. Parameters: a `position` domain, `oneMinusFriction` scaling the tangential component,
`resilience` scaling the reflected normal component, and `cutoffSqr`, a squared tangential
speed below which friction is not applied. Supports plane, rectangle, triangle, disc and
sphere; does nothing for the other kinds. Changes velocity only — never position.

```text
FUNCTION reflect(p, normal, normal_speed)
  vn = normal · normal_speed          # the component along the surface normal
  vt = p.vel - vn                     # the component along the surface
  # Below the cutoff, a particle is sliding rather than skidding: friction would
  # make it creep to a halt, so it is left alone.
  IF |vt|² <= cutoffSqr THEN p.vel = vt - vn · resilience
  ELSE                       p.vel = vt · oneMinusFriction - vn · resilience
```

Planar kinds (plane, rectangle, triangle, disc) share one test:

```text
predict = p.pos + p.vel · dt
skip unless p.pos and predict lie on opposite sides of the plane
for the bounded kinds, find the crossing point and skip unless it lies inside
  the face (barycentric for the triangle, unit square for the rectangle,
  annulus for the disc)
reflect(p, plane normal, p.vel . plane normal)
```

The sphere is different, because it has an interior:

```text
IF the predicted position is inside the shell THEN
  normal = normalize(p.pos - centre)
  IF the current position was ALSO inside THEN
    # Already trapped. Only reverse an inward-pointing velocity, with no
    # restitution loss, so a particle that got inside works its way out
    # instead of rattling forever.
    IF p.vel . normal < 0 THEN p.vel = tangential - normal_component
  ELSE
    reflect(p, normal, p.vel . normal)
```

**Invariants** — the bounce happens a step *before* the crossing, so a particle never visibly
penetrates. It is also never repositioned: the reflection assumes the remaining step is short
enough that the overshoot is invisible, which it is at 33 ms and the speeds effects use.

**Notes** — `oneMinusFriction` is stored pre-subtracted (a value of 1 means frictionless), which
is a micro-optimization in the original and a trap for a rebuild reading authored files: the
number in the file is not the friction coefficient.

The sphere test uses the domain's shell semantics, so an authored inner radius makes a hollow
shell that particles bounce off from both sides.

## `Sink` and `SinkVelocity`

**Contract** — retire particles by testing a domain. `Sink` tests position; `SinkVelocity`
tests velocity, and therefore derives its world-space domain with the emitter's rotation only.
Parameters: the domain and a `kill_inside` flag. Both walk the pool backwards and fire the
death callback.

```text
FOR i FROM pool.live - 1 DOWN TO 0
  inside = domain.within(pool[i].pos)            # or .vel for the velocity variant
  IF inside = kill_inside THEN pool.remove(i)
```

**Notes** — the domain's containment test returns false for the five kinds that have no
interior ([`particle_core.cpp`](particle_core.cpp.md)). A sink pointed at one of those with
`kill_inside` false therefore kills *everything*, every step. The blob domain's test is
random, which turns a sink into a probabilistic thinner — the only way to author one.

## `Restore`

**Contract** — steers every particle back to its second position slot so that it arrives, at
rest, when the countdown expires. Parameter: `time_left`, which the action decrements itself.
Once expired, it pins each particle to the target and zeroes its velocity every step.

```text
FUNCTION execute(pool, dt)
  IF time_left <= 0 THEN
    FOR EACH particle p : p.pos = p.posB ; p.vel = 0
  ELSE
    FOR EACH particle p, per axis
      # Solve for the quadratic velocity profile v(t) that carries the particle from
      # where it is, at the speed it has, to the target at zero speed in time_left,
      # then take one step of it. Evaluated per axis; the three are independent.
      p.vel[axis] = p.vel[axis]
                  + (3·(p.posB - p.pos) - 2·time_left·p.vel[axis]) · (2·dt / time_left²)
                  + (time_left·p.vel[axis] - 2·p.posB + 2·p.pos)   · (3·dt² / time_left³)
  time_left = time_left - dt
```

**Invariants** — the target is the second position slot, so this action is incompatible with
any list that also runs `Move`, which uses that slot as the previous position. An effect using
restore latches the target with `CopyVertexB` (or emits with vertex-B tracking) and then does
its own integration.

**Notes** — the countdown lives in the action, not per particle, so every particle arrives at
the same moment regardless of when it was born. Particles born after the countdown expires are
snapped to their target on their first step.

## `Follow`

**Contract** — accelerates each particle toward the one after it in the pool. Parameters:
`magnitude`, `epsilon`, `max_radius`.

```text
FOR i FROM 0 TO pool.live - 2
  toward = pool[i+1].pos - pool[i].pos ; d2 = |toward|²
  IF max_radius is bounded AND d2 >= max_radius² THEN CONTINUE
  pool[i].vel = pool[i].vel + toward · (magnitude·dt / (sqrt(d2) · (d2 + epsilon)))
```

**Notes** — "the next particle" means the next *slot*, and slots are shuffled by every removal,
so the chain this action seems to describe survives only until the first particle dies. The
authored intent — a ribbon following its leader — therefore works only on effects that never
kill particles. A rebuild should implement the slot semantics, not the intent.

The loop's upper bound is computed as the live count minus one in unsigned arithmetic. On an
empty pool that wraps to an enormous value and the loop runs off the end of the pool. The
caller happens never to run an empty effect's action list at the moment this is reachable; a
rebuild must guard it.

## `Gravitate`

**Contract** — pairwise mutual attraction among all particles, applied symmetrically.
Parameters: `magnitude`, `epsilon`, `max_radius`. Quadratic in the particle count.

```text
FOR EACH pair (i, j), i < j
  toward = pool[j].pos - pool[i].pos ; d2 = |toward|² + tiny
  IF max_radius is bounded AND d2 >= max_radius² THEN CONTINUE
  a = toward · (magnitude·dt / (sqrt(d2) · (d2 + epsilon)))
  pool[i].vel = pool[i].vel + a
  pool[j].vel = pool[j].vel - a          # equal and opposite: momentum is conserved
```

**Notes** — the small constant added to the squared distance keeps a coincident pair from
producing a zero-length direction, separately from `epsilon`, which only softens the
denominator.

## `MatchVelocity`

**Contract** — nominally makes nearby particles agree on a velocity. Parameters: `magnitude`,
`epsilon`, `max_radius`. Quadratic in the particle count.

```text
FOR EACH pair (i, j), i < j
  d2 = |pool[j].pos - pool[i].pos|²
  IF max_radius is bounded AND d2 >= max_radius² THEN CONTINUE
  a = pool[j].vel · (magnitude·dt / (d2 + epsilon))
  pool[i].vel = pool[i].vel + a
  pool[j].vel = pool[j].vel - a
```

**Notes** — what it computes is not a velocity match: it nudges the first particle toward the
second's velocity and nudges the second away from its own by the same amount, which drives
them apart in velocity as often as together. The name records the intent and the arithmetic
records the behaviour; the behaviour is what shipped effects were tuned against.

## `OrbitPoint` and `OrbitLine`

**Contract** — accelerate particles toward a point, or toward the nearest point on a line.
Parameters: the `center` (or the line's `p` and unit `axis`), `magnitude`, `epsilon`,
`max_radius`.

```text
OrbitPoint : toward = center - p.pos
OrbitLine  : f = p.pos - line_point
             toward = axis · (f . axis) - f          # from the particle to the axis

d2 = |toward|²
IF max_radius is bounded AND d2 >= max_radius² THEN skip
p.vel = p.vel + toward · (magnitude·dt / (sqrt(d2) + (d2 + epsilon)))
```

**Notes** — the denominator *adds* the distance to the softened squared distance where the
other attraction actions multiply them. Almost certainly a typo in the original; it makes the
falloff closer to inverse-square than inverse-cube and changes the force scale, and every
authored magnitude for these two actions was tuned against it. Frozen.

Neither action applies a tangential impulse, so "orbit" describes what happens when the action
is combined with an initial tangential velocity, not what the action does.

## `Scatter`

**Contract** — accelerates particles radially away from a centre. Parameters: `center`,
`magnitude`, `epsilon`, `max_radius`.

```text
d = p.pos - center ; d2 = |d|²
IF max_radius is bounded AND d2 >= max_radius² THEN skip
p.vel = p.vel + (d / sqrt(d2)) · (magnitude·dt / (d2 + epsilon))
```

**Notes** — a particle exactly at the centre divides by zero. Emission domains are rarely exact
points, so this is not reached in practice; a rebuild should fall back to no acceleration.

## `Jet`

**Contract** — adds an acceleration *drawn from a domain* rather than a fixed one, attenuated
by distance from a centre. Parameters: a `center`, an `acc` domain, `magnitude`, `epsilon`,
`max_radius`. Consumes one domain sample per affected particle per step, so it is the most
random-hungry action after the source.

```text
d2 = |p.pos - center|²
IF max_radius is bounded AND d2 >= max_radius² THEN skip
p.vel = p.vel + acc.generate() · (magnitude·dt / (d2 + epsilon))
```

**Notes** — the direction to the centre is computed and then used only for its length: the
acceleration's direction comes entirely from the domain, so the centre is a falloff origin, not
a source point. This is what distinguishes jet from scatter.

## `RandomAccel`, `RandomDisplace`, `RandomVelocity`

**Contract** — three ways of applying a domain sample directly. Each carries one domain,
derived with the emitter's rotation only. One sample per particle per step.

```text
RandomAccel    : p.vel = p.vel + gen_acc.generate() · dt
RandomDisplace : p.pos = p.pos + gen_disp.generate() · dt
RandomVelocity : p.vel = gen_vel.generate()            # replaced, not accumulated
```

**Notes** — the first two scale by the step so their authored magnitudes are per-second; the
third deliberately does not, because a velocity is not a rate of anything. The source comment
concedes it is unclear what the step length *should* do to a velocity replacement, and the
answer it settled on — nothing — means an effect using it behaves identically at any step
length, while the other two approach a smooth random walk as the step shrinks.

## `Vortex`

**Contract** — rotates particle *positions* about an axis through a centre, by an angle that
falls off with distance. Parameters: `center`, a unit `axis`, `magnitude`, `epsilon`,
`max_radius`. The only force-like action that moves particles rather than changing their
velocity.

```text
FOR EACH particle p
  offset = p.pos - center ; r2 = |offset|²
  IF max_radius is bounded AND r2 > max_radius² THEN CONTINUE
  r = sqrt(r2) ; n = offset / r
  along  = axis · (n . axis)        # the part of the offset along the axis
  across = n - along                # the part around it
  third  = axis cross across        # completes a frame to rotate in
  theta  = magnitude·dt / (r2 + epsilon)     # tighter near the axis
  p.pos  = center + (across·cos(theta) + third·sin(theta) + along) · r
```

**Invariants** — the distance from the centre is preserved exactly; only the angle changes.
Velocity is untouched, so the move action keeps carrying the particle along its old heading
while the vortex drags it around — the combination is what produces a spiral rather than a
circle.

**Notes** — the bounded and unbounded branches differ only in the range test, and the source's
comment about rejecting particles that are too *close* describes a test that is not there.
Near the axis the angle saturates at `magnitude·dt/epsilon` rather than diverging, which is
what `epsilon` is for here.
