# src/xrPhysics/PHDisabling.cpp

> Decides, once per step per body, whether it has settled enough to stop being
> simulated — using two windows of different lengths so that neither noise nor slow drift
> fools it.

**Needs** — [`PHDisabling.h`](PHDisabling.h.md) · [`DisablingParams.h`](DisablingParams.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHDisabling.h`](PHDisabling.h.md)
**Tier floor** — T2: arithmetic on two accumulators per body per step.

## Purpose

Sleeping is not an optimization here, it is a correctness mechanism: a body that is never
disabled keeps receiving solver jitter and slowly walks across the floor, and a hundred of
them exhaust the frame budget. But disabling too eagerly freezes an object mid-fall.

The design answers that with **two tests of different length**, and the interesting content
of this file is why one window is not enough.

- The **short window** is one step. It asks: is this step's displacement and this step's
  velocity change small? It catches a body that has genuinely stopped, immediately.
- The **long window** is 64 steps. It sums the per-step *changes* over the window and
  divides by the window length. Its crucial property is that it sums **signed** changes, so
  a body vibrating back and forth — which passes no single short-window test but is going
  nowhere — sums to nearly zero and is caught. Equally, a body creeping in one direction at
  a speed below the short threshold accumulates and is *not* caught, which is the case a
  single short test would wrongly put to sleep.

## State

```text
RECORD DisableAccumulator            # one per measured quantity
  previous : vector                  # last sample
  sum      : vector                  # signed sum of per-step deltas since window start

RECORD WindowVerdict
  may_sleep  : bool                  # motion was under threshold
  must_wake  : bool                  # motion was over threshold * reenable_factor
  # these are NOT complements: between the two lies the hysteresis band, where a
  # sleeping body stays asleep and a moving body stays awake.

RECORD DisableState
  count              : int (16-bit)  # steps remaining until the long window closes
  frames             : int (16-bit)  # long window length, from the object's settings
  last_step_updated  : int (16-bit)  # the step this was last run for
  short_verdict      : WindowVerdict
  long_verdict       : WindowVerdict
  disabled           : bool
  # invariant: count is in [0, frames]; reaching 0 closes the long window and reloads it
```

## the once-per-step driver

**Contract** — run the whole decision for one body. Idempotent within a step: called twice
in the same step, the second call does nothing.

```text
FUNCTION disabling(self)
  IF the world is frozen               RETURN      # activation procedures run here; don't judge
  IF last_step_updated == world.short_step_count   RETURN
  last_step_updated = world.short_step_count
  count = count - 1

  update_short_window()                # sample, and set short_verdict
  apply(short_verdict)

  IF count == 0
    update_long_window()               # fold the accumulated sums into long_verdict,
    apply(long_verdict)                #   then reset the accumulators
    count = frames

  # Any external force or torque standing on the body vetoes sleep outright.
  IF the body carries a non-zero force or torque
    disabled = false

  IF the body is currently awake
    re_enable()                        # the owner's hook: re-arm whatever sleep switched off
    IF NOT disabled AND our phase differs from the world's
      count = frames + world.disable_phase     # re-stagger, see Notes

  IF disabled
    disable()                          # the owner's hook: actually stop simulating

FUNCTION apply(verdict)
  IF disabled     disabled = NOT verdict.must_wake     # stay asleep unless clearly moving
  ELSE            disabled = verdict.may_sleep         # fall asleep only if clearly still
```

**Invariants** — the world's frozen state suppresses the whole test, because the activation
procedures in this chapter deliberately run dozens of solver steps with unphysical
velocities and must not have their subjects fall asleep mid-correction.

**Notes** — the asymmetric `apply` is the hysteresis in action, and it is the difference
between "objects settle" and "objects flicker". Read it as a two-state machine where the
thresholds for the two transitions differ by the re-enable factor.

The force veto is a hard override rather than another window, and it has to be: a body held
against a wall by a script-applied force is motionless by every measure, and putting it to
sleep would make the force stop taking effect.

The **phase staggering** is the file's most obscure decision and the one a rebuilder will
otherwise mangle. Every body's long window is 64 steps, so if all bodies started their
window on the same step, all of them would run the expensive long test on the same step and
the frame cost would spike every 64 steps. The world keeps a rotating phase counter; each
body, when it wakes, offsets its window by the world's current phase, which spreads the long
tests uniformly. It changes no outcome — only when each body's outcome is computed.

## the accumulators

**Contract** — each step, an accumulator is handed the current sample. It returns the
magnitude of the change since the previous sample and folds that change, **signed**, into
its running sum. A second entry point updates the previous sample without folding, used when
the long window has already closed and the sum must not grow further.

```text
FUNCTION update(acc, sample) -> real
  delta        = sample - acc.previous
  acc.previous = sample
  acc.sum      = acc.sum + delta       # signed: oscillation cancels
  RETURN |delta|
```

## the threshold comparison

**Contract** — given a velocity measure and an acceleration measure, set the verdict:

```text
FUNCTION check(verdict, velocity, acceleration, thresholds)
  IF velocity < thresholds.velocity AND acceleration < thresholds.acceleration
    verdict.may_sleep = true                     # BOTH must be small
  IF velocity     > thresholds.velocity     * reenable_factor
  OR acceleration > thresholds.acceleration * reenable_factor
    verdict.must_wake = true                     # EITHER being large is enough
```

**Notes** — the two conditions use `and` and `or` respectively, which is the pessimistic
choice in both directions: hard to fall asleep, easy to wake. A body must be both slow and
unaccelerating to sleep; it wakes on either.

## what each window measures

**Contract** — the short window scales its per-step deltas by the window length before
comparing, so that both windows are compared against the *same* thresholds. Without that,
the two windows would need two sets of numbers.

The **translational** form samples the body's position and its linear velocity. The
**rotational** form samples the body's angular velocity and — instead of an angle — three
specific off-diagonal components of its orientation matrix.

**Notes** — using three matrix components as a stand-in for orientation is the file's one
genuinely arbitrary-looking choice, and the reason is that it is *cheap and monotonic for
small rotations*: for a body that is nearly still, the change in those components is
proportional to the rotation angle, and no trigonometry or quaternion difference is needed.
It breaks down for large rotations, which is exactly the case where the body is obviously
awake and the measure does not matter. A rebuild may use any small-angle proxy; it must not
use an Euler-angle difference, which wraps.

The identity of the three components is not otherwise meaningful — they are one entry from
each of the three off-diagonal pairs — and this is the clearest case in the chapter of a
constant whose specific value could not be recovered from the source.

## the combined form

**Contract** — runs both the rotational and the translational test and combines their
verdicts: may-sleep only if **both** agree, must-wake if **either** objects.

**Notes** — a spinning body sliding to a halt and a still body slowly rotating must both
stay awake, which is exactly what this gives. The combination is applied independently to
the short and long windows, so a body can be short-window still and long-window moving.
