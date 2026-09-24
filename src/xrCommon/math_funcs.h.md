# src/xrCommon/math_funcs.h

> The angle arithmetic the whole engine turns with: normalization to a chosen range, shortest-way-round difference, and the two rate-limited "turn toward" loops.

**Needs** — [`xrCore/math_constants.h`](../xrCore/math_constants.h.md) · [`math_funcs_inline.h`](math_funcs_inline.h.md) · [`utils/xrMiscMath/vector.cpp`](../utils/xrMiscMath/vector.cpp.md)
**Used by** — [`vector.h`](../xrCore/vector.h.md)
**Tier floor** — T3: pure scalar arithmetic. Nothing here touches a device or a byte layout.

## Purpose

An angle in this engine is a plain number of radians with no type of its own, which means
every caller must decide, per use, which range it is in and what "the difference between
two angles" means. This file makes those decisions once. It is the reason a creature turns
the short way round instead of spinning 350 degrees, and the reason a camera that has been
turning for ten minutes still compares correctly against a fresh target.

The declarations are here and the bodies are in
[`utils/xrMiscMath/vector.cpp`](../utils/xrMiscMath/vector.cpp.md) — a split of no
significance; the contracts on this page are what a rebuild needs, and the numeric detail
below is part of the contract because callers depend on the exact boundary behaviour.

## `angle_normalize_always`

**Contract** — maps any angle to the half-open range from zero up to but not including one
full turn. Unconditional: it does the arithmetic even when the input is already in range.

```text
FUNCTION angle_normalize_always(a) -> real
  turns = a / FULL_TURN
  whole = truncate turns toward zero        # NOT floor: the sign is handled below
  frac  = turns - whole                     # in (-1, 1)
  IF frac < 0
    frac = frac + 1
  RETURN frac * FULL_TURN
```

**Invariants**

```text
# Result is in [0, FULL_TURN). Exactly zero maps to zero; one negative full turn
# maps to zero, not to FULL_TURN.

# The whole-turns count passes through a 32-bit signed integer.
#   An angle beyond about two billion turns is undefined. Reachable only by a
#   runaway accumulator, which is the bug this would be masking anyway.

# Precision degrades with magnitude, because the fraction is taken after the
# division. Angles that have been accumulating all session lose low bits. Nothing
# in the engine periodically re-normalizes its stored angles, so a rebuild that
# does so is CHANGING behaviour, not fixing it - a stored camera angle and an
# orientation matrix derived from it must stay consistent.
```

## `angle_normalize`

**Contract** — the same range, but returns the input untouched when it already lies between
zero and one full turn inclusive. This is a fast path, and it is not quite the same
function: an input of exactly one full turn is returned as itself, which the unconditional
version would have mapped to zero. Callers that compare the result against zero must not
use this one.

## `angle_normalize_signed`

**Contract** — maps any angle to the range from minus half a turn to plus half a turn,
returning the input untouched when it is already in that range. This is the canonical form
for anything that will be *differenced*: a heading, a target bearing, a joint limit.

```text
FUNCTION angle_normalize_signed(a) -> real
  IF a is within [-HALF_TURN, HALF_TURN]
    RETURN a
  angle = angle_normalize_always(a)         # now in [0, FULL_TURN)
  IF angle > HALF_TURN
    angle = angle - FULL_TURN
  RETURN angle
```

## `angle_difference_signed`

**Contract** — the signed shortest rotation that takes the second angle to the first,
always at most half a turn in magnitude. Positive means one direction, negative the other,
consistently. This is the single most-used function in this file: every "turn toward"
decision in the engine is the sign of this value, and every "am I aimed at it" test is its
magnitude.

```text
FUNCTION angle_difference_signed(a, b) -> real
  diff = angle_normalize_signed(a) - angle_normalize_signed(b)
  IF diff > HALF_TURN
    diff = diff - FULL_TURN
  ELSE IF diff < -HALF_TURN
    diff = diff + FULL_TURN
  RETURN diff
```

**Invariants**

```text
# Result is in [-HALF_TURN, HALF_TURN].
# Exactly half a turn apart is a tie and resolves to the positive direction,
# because the wrap tests are strict. A creature facing exactly away from its
# target always turns the same way; this is arbitrary but must be deterministic,
# since the multiplayer server and client both compute it.
```

## `angle_difference`

**Contract** — the magnitude of the above: the unsigned shortest angle between two
directions, from zero to half a turn. Used for "how far off am I" thresholds.

## `are_ordered` / `is_between`

**Contract** — `are_ordered` answers whether the middle value lies between the outer two,
*in either direction* — the outer pair is an unordered interval, and the bounds are
inclusive. `is_between` is the same question with the value written first, for readability
at the call site.

**Notes** — these are plain scalar range tests, not angle-aware: they know nothing about
wrapping. They live in this file only because the turn loop below uses one of them.

## `angle_lerp` (rate-limited, in place)

**Contract** — advances a current angle toward a target by at most a given rate multiplied
by an elapsed time, moving the short way round, and reports whether the current angle was
*already* at the target. The current angle is updated in place. Never overshoots: when the
allowed step exceeds the remaining difference, the result is the target exactly.

```text
FUNCTION angle_lerp(current, target, rate, elapsed) -> bool   # current is updated
  before = current
  diff = target - current
  IF diff > HALF_TURN                      # take the short way round
    diff = diff - FULL_TURN
  ELSE IF diff < -HALF_TURN
    diff = diff + FULL_TURN

  remaining = magnitude of diff
  IF remaining < EPS_TINY
    RETURN true                            # already there; current untouched

  step = rate * elapsed
  IF step > remaining
    step = remaining                       # land exactly, never overshoot
  current = current + (sign of diff) * step

  IF current lies between `before` and `target`
    RETURN false                           # no wrap happened; leave as is

  IF current < 0
    current = current + FULL_TURN
  ELSE IF current > FULL_TURN
    current = current - FULL_TURN
  RETURN false
```

**Invariants**

```text
# The return value means "was already at the target", not "arrived this step".
#   A step that lands exactly on the target still returns false; the caller sees
#   true only on the following call. Turn loops that treat the result as "arrived"
#   therefore run one extra frame. This is the observable behaviour and a rebuild
#   should keep it, because animation state machines are tuned against it.

# The arrival threshold is the tiniest epsilon (see math_funcs_inline.h), not the
# ordinary one. At that scale the test is effectively exact equality, which is
# why the extra frame above exists at all.

# The re-wrap is SKIPPED when the new value is still between where it started and
# the target.
#   Only a step that crossed the wrap boundary needs fixing up. This is why the
#   direction-agnostic range test above is in this file: it is the boundary
#   detector, not a convenience.

# The target is expected already normalized. Passing an unnormalized target makes
# the short-way-round correction pick the wrong direction. Every caller normalizes
# first; nothing checks.
```

## `angle_lerp` (fraction)

**Contract** — interpolates between two angles by a fraction, going the short way round.
Both inputs are expected normalized to one turn. A fraction of zero gives the first angle,
one gives the second; the result is *not* re-normalized, so interpolating near the wrap
boundary legitimately returns a value slightly outside the range and the caller normalizes
if it cares.

## `angle_inertion`

**Contract** — a *leashed* follow: advance a source angle toward a target at a given rate,
then force the result to lag the target by no more than a given limit. Returns the new
angle; the source is not modified. This is how a creature's head follows its gaze and how a
torso follows its head — the part turns smoothly but can never be further from the target
than its anatomy allows.

```text
FUNCTION angle_inertion(source, target, rate, limit, elapsed) -> real
  target = angle_normalize_signed(target)
  angle_lerp(source, target, rate, elapsed)       # source advances in place
  source = angle_normalize_signed(source)
  lag       = angle_difference_signed(source, target)
  lag_capped = clamp lag into [-limit, limit]
  RETURN source - (lag - lag_capped)              # snap back onto the leash
```

**Notes** — the leash is applied *after* the rate limit, so a target that jumps further
than the limit drags the source with it instantly. That is deliberate: the limit expresses
a physical constraint (how far a head can turn relative to a body), and a constraint must
never be violated even for one frame, whereas the rate only expresses smoothness.

## `angle_inertion_var`

**Contract** — the same leashed follow, with the rate chosen per call: it scales linearly
from a minimum rate when the source is already on target to a maximum rate when the source
is a full leash-length away. The effect is that a part snaps quickly when badly misaligned
and creeps when nearly aligned, which reads as attention rather than as machinery.

```text
FUNCTION angle_inertion_var(source, target, rate_min, rate_max, limit, elapsed) -> real
  target = angle_normalize_signed(target)
  source = angle_normalize_signed(source)
  off  = angle_difference(target, source)             # 0 .. HALF_TURN
  rate = magnitude of ((rate_max - rate_min) * off / limit) + rate_min
  ... then exactly as angle_inertion, with this rate
```

**Invariants**

```text
# rate_max is NOT a cap.
#   The interpolation is unclamped, so an offset larger than the leash length
#   produces a rate above the maximum - unboundedly so, since the offset can be
#   half a turn while the leash is a few degrees. Callers whose leash is small
#   relative to a plausible offset get an effectively instant snap. Every shipped
#   tuning pair is read from configuration, so a rebuild that "fixes" this by
#   clamping will change the feel of creatures whose data relies on the snap.
```
