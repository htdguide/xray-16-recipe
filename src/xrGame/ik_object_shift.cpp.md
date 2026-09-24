# src/xrGame/ik_object_shift.cpp

> Moves a character's whole body vertically along a smooth cubic so its feet can reach the ground, with an overshoot guard that degrades to a parabola when the cubic would bounce.

**Needs** — [`ik_object_shift.h`](ik_object_shift.h.md) · [`pose_extrapolation.h`](pose_extrapolation.h.md) · [`xrPhysics/MathUtils.h`](../xrPhysics/MathUtils.h.md)
**Used by** — [`ik_object_shift.h`](ik_object_shift.h.md)
**Tier floor** — T2: polynomial trajectory arithmetic

## Purpose

A foot on a step below the character cannot be reached by bending the leg alone; the body
must come down. Doing that abruptly is worse than not doing it, so the shift is a *trajectory*
rather than a value: given a target and a time to reach it, the smoother builds a curve that
arrives at the target **with zero velocity** and starts from wherever the previous curve had
got to, at whatever speed it was moving.

That continuity requirement is what makes this file non-trivial. A new target can arrive at
any moment, from a foot that has just discovered a different step, and the new curve must
start where the old one is without a visible kink.

## State

```text
RECORD ObjectShift
  current, target           : real
  current_time, target_time : real
  speed, accel, aaccel      : real   # the polynomial's 1st, 2nd and 3rd derivatives at
                                     # `current_time`
  frozen                    : bool
```

**Invariants** — the trajectory is defined entirely by the six numbers. There is no stored
curve and no per-frame integration: querying the shift evaluates the polynomial at the
current time, so the value is the same no matter how often it is asked for or how the frame
rate varies.

## The trajectory

```text
FUNCTION delta(dt) -> real
  IF frozen THEN RETURN 0
  RETURN speed*dt + accel*dt^2/2 + aaccel*dt^3/6
```

A cubic: constant jerk, with the three derivatives stated at the trajectory's own start time.

## `shift`

**Contract** — the shift value now. Time is **clamped at the target time**, so once the
trajectory arrives it holds rather than continuing past.

```text
FUNCTION shift() -> real
  t = min(now, target_time)
  value = current + delta(t - current_time)
  RETURN clamp(value, -max_shift, +max_shift)
```

**Invariants** — the clamp at the target time is essential: a cubic that reaches its target
with zero velocity still has non-zero acceleration there and would immediately curve away.
Freezing the evaluation at the arrival time is what makes the trajectory terminate.

```text
max_shift = 1.0    # metres, in each direction
```

The hard limit exists because a bad ground query — a hole in the collision geometry, a
surface a metre below the floor — would otherwise drop the character's body arbitrarily far.
One metre is about the deepest real step in the shipped levels.

## `set_taget`

**Contract** — aims for a new value at a time in the future, preserving position and velocity
continuity with whatever the current trajectory is doing. Refused outright while frozen.

```text
FUNCTION set_target(new_target, duration)
  IF frozen THEN RETURN
  new_target = clamp(new_target, -max_shift, +max_shift)
  target = new_target

  # 1. Already going there? Just extend the deadline.
  IF the existing trajectory evaluated at (now + duration) already equals the target THEN
    target_time = now + duration
    RETURN

  # 2. Sample where the current trajectory is, in value and in rate.
  IF duration is effectively zero THEN duration = one frame
  current = shift()                       # position continuity
  elapsed = (now > target_time) ? (target_time - current_time)     # the old curve ENDED
                                : (now - current_time)
  speed = speed + accel*elapsed + aaccel*elapsed^2/2               # rate continuity

  # 3. Solve a cubic that travels x in `duration` and ARRIVES AT REST.
  x = target - current
  accel  =  2 * (3*x/duration^2 - 2*speed/duration)
  aaccel =  6 * (speed/duration^2 - 2*x/duration^3)

  # 4. Overshoot guard — see below
  current_time = now
  target_time  = now + duration
```

**Invariants**, each of which a rebuild must reproduce:

- **The elapsed time is clamped to the old trajectory's end.** If the old curve had already
  arrived, the sampled velocity is its velocity *at arrival* — which the construction made
  zero — not the velocity of a polynomial extrapolated past its domain. Miss this and a
  target set after a period of rest starts with a large spurious velocity.
- **The new curve is fully determined by four conditions**: start at the sampled position with
  the sampled velocity, end at the target with zero velocity. Two of those are absorbed into
  the initial state and two fix the second and third derivatives. The zero terminal velocity is
  what makes successive shifts compose without jitter.
- **A zero duration becomes one frame.** A shift is never instantaneous, because an
  instantaneous body shift is the artifact the whole file exists to prevent.
- **The early exit extends the deadline without rebuilding the curve.** A foot reporting the
  same ground it reported last frame must not restart the trajectory every frame, or the
  shift never converges — each restart resets the remaining duration and the body creeps
  toward the target forever.

## The overshoot guard

**Contract** — a cubic that satisfies all four boundary conditions is not necessarily monotone:
for a large enough move with an inconvenient starting velocity it swings well past the target
and comes back. That reads as a visible bounce in the character's height. The guard detects it
and substitutes a curve that cannot bounce.

```text
# only for moves large enough to be visible
IF x > 0.1 (moving up) OR x < -0.15 (moving down) THEN
  # the trajectory's velocity is a quadratic; its roots are its turning points
  IF (aaccel/2)*t^2 + accel*t + speed = 0 has real roots t0, t1 THEN
    FOR EACH root INSIDE (0, duration)
      IF |delta(root)| > 2*|x| THEN          # it swings past by more than the move itself
        # fall back to a PARABOLA: still arrives at x with zero velocity, but monotone
        aaccel = 0
        speed  = 2*x / duration              # velocity continuity is SACRIFICED
        accel  = -speed / duration
        BREAK
```

**Invariants**:

- **The thresholds are asymmetric**: a hundredth-and-a-bit of a metre upward, half again as
  much downward. Moving a character's body *up* too fast is more visible than moving it down —
  an upward bounce reads as a hop — so the guard engages earlier for upward moves. The two
  numbers are empirical and are the kind of constant a rebuild will have to re-tune against
  its own animation set.
- **The overshoot criterion is "twice the move", not "past the target".** A small overshoot is
  natural-looking and is allowed; only a swing of more than double the intended distance is
  treated as a bounce.
- **The fallback breaks velocity continuity.** The parabolic profile starts at a velocity of
  twice the average, which will generally not match the velocity the old curve was moving at.
  That is the deliberate trade: a single small kink at the moment of retargeting is less
  visible than a bounce spread over the whole move. A rebuild that tries to preserve both will
  find the problem is over-constrained.
- Below the thresholds the guard is skipped entirely, so small corrections keep their cubic
  and their continuity.

## `reset`

**Contract** — sets the shift to zero at the current instant with all three derivatives zero
and the target time equal to now, which means the trajectory has already arrived. Used when a
character is teleported, respawned, or otherwise has no continuity worth preserving.

## `freeze`

**Contract** — while frozen, the trajectory contributes nothing (the delta is zero) and new
targets are refused. The stored state is left intact, so unfreezing resumes from a trajectory
whose start time is now in the past — which the evaluation handles, because it is a polynomial
in elapsed time and the clamp at the target time bounds it.

## `square_equation`

**Contract** — the real roots of a quadratic, or nothing when the discriminant is negative.

**Notes** — it divides by the leading coefficient without testing it for zero. The caller
passes half the jerk, which is zero exactly when the trajectory is already a parabola — and a
parabola cannot bounce, so the guard has nothing to do. A rebuild should test it anyway.

## Debug drawing

**Notes** — a compiled-out visualization samples the trajectory at two hundred points and draws
it against the pose extrapolation's predicted position, which is how the overshoot guard's
thresholds were arrived at. Not part of the behaviour.
