# src/xrPhysics/DisablingParams.cpp

> The default sleep thresholds and the per-object overrides read from
> configuration.

**Needs** — [`DisablingParams.h`](DisablingParams.h.md) · [`xrCore`](../xrCore/README.md)
**Used by** — [`DisablingParams.h`](DisablingParams.h.md)
**Tier floor** — T3: parsing four values out of a text section.

## Purpose

A rigid-body world that does not put settled objects to sleep spends its whole budget
re-solving a hundred motionless crates. The sleep decision is in
[`PHDisabling.cpp`](PHDisabling.cpp.md); the *numbers* it decides against are here, so they
can be overridden per model from the shipped configuration without touching the algorithm.

## State

```text
RECORD OneAxisThresholds
  velocity     : real    # motion below this over the window counts as still
  acceleration : real    # change of velocity below this counts as unforced

RECORD ObjectThresholds
  translational : OneAxisThresholds
  rotational    : OneAxisThresholds
  long_window   : int (16-bit)   # length of the long test window, in physics steps

RECORD WorldThresholds
  object_defaults  : ObjectThresholds
  reenable_factor  : real        # how far above threshold motion must rise to WAKE
```

**Invariants** — the long window must never be zero; it is a divisor.

## the shipped defaults

**Contract** — every object starts from one shared set of values:

```text
translational : velocity 0.001, acceleration 0.1
rotational    : velocity 0.005, acceleration 0.05
long_window   : 64 steps
reenable      : 1.5
```

**Notes** — the ratios are the interesting part, and they were tuned rather than derived.
Rotation tolerates five times more residual velocity than translation, because the
rotational measure is taken from matrix components rather than an angle (see
[`PHDisabling.cpp`](PHDisabling.cpp.md)) and is noisier for the same physical stillness.
Translation tolerates twice the residual acceleration of rotation, because a resting body on
a slope carries a real, permanent gravity-versus-friction imbalance that is not motion.

The window of 64 steps is a little over a second of simulated time at the shipped step rate.
It is the single knob that trades "objects visibly keep twitching" against "objects freeze
while still settling".

`reenable_factor` of 1.5 is a **hysteresis band**, and it is the reason objects do not
chatter between asleep and awake: a sleeping body needs motion half again above the
threshold before it wakes, so a body sitting exactly at the threshold stays put. A rebuild
that uses one threshold for both directions will produce visible flickering on slopes.

## `SAllDDOParams::Load`

**Contract** — reads overrides from a configuration section named `disable` on the object's
own settings, starting from the world defaults:

```text
FUNCTION load(settings)
  this = world_defaults
  IF no "disable" section          RETURN
  IF "linear_factor"  present      scale BOTH translational thresholds by it
  IF "angular_factor" present      scale BOTH rotational thresholds by it
  IF "change_count"   present
    n = signed value
    long_window = long_window shifted left by n, or right by -n when n is negative
```

**Invariants** — the shift count must be under 4 and the window must not shift to zero;
violating either is an authoring error, checked only in the debug build. A shipped
configuration that shifts the window to zero would divide by zero at the next step.

**Notes** — overrides are *multiplicative factors and power-of-two shifts*, never absolute
values. That is a deliberate authoring decision: a modder saying "this object should settle
twice as easily" writes `2`, and the relationship between the four thresholds — which was
tuned together — is preserved. Absolute values would let one axis drift out of proportion
with the others, which is exactly the tuning failure the ratios above are protecting.
