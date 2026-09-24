# src/utils/xrMiscMath/vector.cpp

> Angle arithmetic on the circle, and the out-of-line body of the 3-vector.

**Needs** — [`xrCore/_vector3d.h`](../../xrCore/_vector3d.h.md) · [`xrCore/_random.h`](../../xrCore/_random.h.md) · [`xrCore/math_constants.h`](../../xrCore/math_constants.h.md) · [`xrCommon/math_funcs_inline.h`](../../xrCommon/math_funcs_inline.h.md) · [`xrCore/_bitwise.h`](../../xrCore/_bitwise.h.md) · [`xrCore/xrDebug.h`](../../xrCore/xrDebug.h.md) · [Platform assumptions](../../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)
**Used by** — [`xrMiscMath.cpp`](xrMiscMath.cpp.md) · [`math_funcs.h`](../../xrCommon/math_funcs.h.md) · [`_vector3d.h`](../../xrCore/_vector3d.h.md)
**Tier floor** — T2: pure arithmetic, but three vectors per bone per frame and the layout is a foreign-boundary contract (vertex streams, physics bodies), so boxing small aggregates is not an option

## Purpose

Two unrelated things share this file. The first is the engine's entire treatment of
**angles as points on a circle** — normalization into a chosen interval, signed shortest
difference, and rate-limited interpolation toward a target. Every heading in the game is a
bare real number in radians, so the wrap-around rules live here and nowhere else. The
second is the out-of-line half of the 3-vector: the operations the header declared but did
not inline.

The pairing is arbitrary and a rebuild should split it: the angle routines have no
dependency on the vector type, and the vector bodies have no dependency on the angle
routines.

## State

`Stateless.` The random-direction routines draw from a caller-supplied generator; nothing
in this file holds a value between calls.

## Angles — the interval contract

The engine uses two intervals and never mixes them within one routine:

- **unsigned**, `[0, 2π]`, for stored and authored headings;
- **signed**, `[-π, π]`, for *differences* and for anything fed to interpolation.

A rebuilder must carry both. Collapsing to one interval breaks either the serialized
headings or the shortest-path logic.

## `angle_normalize_always`

**Contract** — Maps any real angle into `[0, 2π)`. Total, no fast path.

```text
FUNCTION angle_normalize_always(a) -> real
  turns = a / TWO_PI
  # truncate toward zero, then repair the sign, rather than take a remainder:
  # a remainder operation keeps the sign of its left operand, so a negative
  # angle would come back negative; the repair below is what forces [0, 2PI)
  whole = IF turns > 0 THEN floor(turns) ELSE ceil(turns)
  frac  = turns - whole
  IF frac < 0 THEN frac = frac + 1
  RETURN frac * TWO_PI
```

## `angle_normalize`

**Contract** — The same map, with a fast path that returns the input untouched when it is
already in range. Total.

**Notes** — The fast path's range test is inclusive at both ends, so an input of exactly
`2π` is returned as `2π` rather than `0`. The stated interval is therefore closed, not
half-open, whenever the fast path fires — a rebuild that normalizes strictly will differ
on that one value. Nothing in the engine depends on which way it goes, but a rebuilder
comparing against the original will see it.

## `angle_normalize_signed`

**Contract** — Maps any real angle into `[-π, π]`, with the same inclusive fast path.
Total.

## `angle_difference_signed`

**Contract** — The shortest signed rotation that takes `b` to `a`, in `[-π, π]`. Normalizes
both inputs, subtracts, then folds the result back across the half-turn boundary. Total.

**Invariants** — Sign convention: positive means `a` is counterclockwise of `b` in the
engine's angle direction. Every "turn toward" decision in the AI and camera layers reads
this sign.

## `angle_difference`

**Contract** — The magnitude of the above, in `[0, π]`. This is the engine's angular
distance: the thing compared against field-of-view half-angles and aim tolerances.

## `are_ordered` and `is_between`

**Contract** — `are_ordered(a, b, c)` is true when `b` lies in the closed interval spanned
by `a` and `c`, *in either direction* — the interval is unordered. `is_between` is the same
predicate with the value first. Both total, both on plain reals with no angle wrapping.

**Notes** — Accepting the interval in either direction is the load-bearing part: callers
pass a start and an end that may be given in either order, and asking them to sort first
is exactly the bug this exists to prevent.

## `angle_lerp(current, target, speed, dt)` — rate-limited turn

**Contract** — Advances `current` toward `target` by at most `speed × dt`, taking the
shorter way around, and reports whether the turn is already complete. Mutates its first
argument. Returns true when the remaining difference was below the tightest epsilon
*before* stepping — that is, "arrived, nothing to do" — and false in every other case,
including the case where this call lands exactly on the target.

```text
FUNCTION turn_toward(current, target, speed, dt) -> bool   # current is updated in place
  before = current
  diff = target - current
  # fold to the shorter way around; inputs are assumed already normalized
  IF diff > PI        THEN diff = diff - TWO_PI
  ELSE IF diff < -PI  THEN diff = diff + TWO_PI

  IF abs(diff) < TIGHTEST_EPSILON
    RETURN true                         # already there

  step = min(speed * dt, abs(diff))
  current = current + sign(diff) * step

  IF is_between(current, before, target)
    RETURN false                        # still inside the span we started in: no wrap possible
  # the step crossed the seam, so pull the result back into [0, 2PI]
  IF current < 0        THEN current = current + TWO_PI
  ELSE IF current > TWO_PI THEN current = current - TWO_PI
  RETURN false
```

**Notes** — The wrap repair is skipped when the new value still lies between where it
started and the target, which is the common case and cannot have left the interval. This
is an optimization, not a semantic difference, but it is worth reproducing because it also
means the routine never renormalizes an already-in-range value and so never introduces
rounding on the hot path.

## `angle_lerp(a, b, t)` — plain interpolation

**Contract** — Blends between two angles along the shorter arc at fraction `t`. Expects
both inputs already in `[0, 2π)`; does not normalize, does not clamp `t`, and may return a
value outside the interval. Total.

## `angle_inertion(src, tgt, speed, clamp, dt)`

**Contract** — One step of a *lagging* follow: turn toward the target at the given rate,
then force the result to be no further than `clamp` radians behind. Returns the new angle
in the signed interval.

```text
FUNCTION inertion(src, tgt, speed, limit, dt) -> real
  tgt = to_signed(tgt)
  turn_toward(src, tgt, speed, dt)
  src = to_signed(src)
  lag = angle_difference_signed(src, tgt)
  # the follower is allowed to trail, but never by more than `limit`;
  # whatever lag exceeds the limit is removed at once
  src = src - (lag - clamp_to(lag, -limit, +limit))
  RETURN src
```

**Notes** — This is how a head, a weapon or a camera trails what it is tracking: the rate
gives the smooth motion, the clamp guarantees the follower is never more than a fixed
angle off no matter how fast the target moved. Both halves are needed; a rate alone lets a
teleporting target leave the follower arbitrarily far behind, and a clamp alone snaps.

## `angle_inertion_var(src, tgt, min_speed, max_speed, clamp, dt)`

**Contract** — The same, with the turn rate interpolated linearly between `min_speed` and
`max_speed` in proportion to how far off the follower currently is, measured against the
clamp as the full-scale error. Larger error, faster turn.

**Invariants** — The clamp doubles as the error scale, so it may not be zero.

## `rsqrt`

**Contract** — Reciprocal square root in double precision. Total for positive inputs.

**Notes** — Despite the name this is an exact `1 / sqrt(v)`, not a fast approximation. The
name is a fossil from a version that used one. Its one caller is the robust normalize in
[`xrMiscMath.cpp`](xrMiscMath.cpp.md), which needs the accuracy, not the speed — so a
rebuild must **not** substitute a fast reciprocal here.

## 3-vector — magnitude and normalization

**Contract** — `square_magnitude` and `magnitude` are the obvious sums; `distance_to`,
`distance_to_sqr`, `distance_to_xz` and `distance_to_xz_sqr` are the same over a
difference. Total.

**Invariants** — The `xz` variants drop the vertical component. Horizontal distance is a
first-class operation throughout the game layer because the world is walked on a
mostly-horizontal plane: "is this creature within ten metres" almost always means ten
metres of ground distance.

The normalization family is four routines and the differences are the whole point:

| routine | zero-length input | reads from |
|---|---|---|
| `normalize` | asserted against | itself |
| `normalize_safe` | left unchanged | itself |
| `normalize(v)` | asserted against | argument |
| `normalize_safe(v)` | destination left unchanged | argument |
| `normalize_magn` | asserted against | itself, and returns the old length |

```text
FUNCTION normalize_in_place(v)
  m2 = v.x*v.x + v.y*v.y + v.z*v.z
  REQUIRE m2 > smallest_positive_real        # checked only in instrumented builds
  # note the shape: one reciprocal then one root, not a root then a reciprocal.
  # mathematically identical, and the engine relies on neither -- but it does
  # mean a zero vector produces an infinity, not a division fault
  s = sqrt(1 / m2)
  v = v scaled by s
```

**Notes** — The guard is against the smallest *normal* magnitude, not against an epsilon.
A vector whose components are around 1e-20 therefore fails this guard even though it has a
perfectly good direction; recovering that direction is what
[`exact_normalize`](xrMiscMath.cpp.md) exists for. The "safe" variants are not more
accurate, only non-fatal: they leave a too-small vector exactly as it was, which means the
caller silently keeps a non-unit vector. Every caller of a safe variant is therefore
accepting a possibly-unnormalized result.

## 3-vector — blending and accumulation

**Contract** — `lerp(p1, p2, t)` blends two vectors; `average` takes the midpoint of two
or of self-and-one; the four `mad` overloads are the multiply-accumulate family, with the
multiplier either a scalar or a per-component vector and the base either the current value
or a supplied one. None clamp, none allocate, all total.

**Invariants** — `inertion(p, v)` weights *itself* by `v` and the argument by `1 - v`. The
parameter is a retention factor, not a blend toward `p`: passing 1 keeps the current value
and passing 0 snaps to `p`. This is the reverse of the reading most people take from the
name, and the tuning values in the game data are authored against this sense.

## 3-vector — shaping

**Contract** — `set_length(l)` rescales to a given length (undefined for a zero vector — no
guard). `squeeze(e)` flushes each component whose magnitude is below `e` to exactly zero.
`clamp(min, max)` clamps per component against a box; `clamp(v)` clamps per component
against the symmetric box `±|v|`. `align()` snaps to the nearest horizontal axis direction.

```text
FUNCTION align(v)                      # snap to one of +X, -X, +Z, -Z
  v.y = 0
  IF abs(v.z) >= abs(v.x)
    v.z = sign_or_zero(v.z)            # the guard makes an all-zero vector
    v.x = 0                            # come out as zero rather than a fault
  ELSE
    v.x = sign(v.x)
    v.z = 0
```

**Notes** — `squeeze` exists to stop tiny residues accumulating in values that get
serialized and compared — a component left at 1e-30 by a chain of rotations is noise that
makes an otherwise identical transform compare unequal, and on some targets it is also a
denormal that costs hundreds of cycles to touch.

## 3-vector — geometry

**Contract** — `crossproduct(v1, v2)` writes the right-handed cross product into the
destination, which must not alias either operand. `mknormal_non_normalized(p0, p1, p2)` is
the cross of the two edges of a triangle taken in winding order; `mknormal` is the same
followed by a *safe* normalize, so a degenerate triangle yields the zero vector rather than
a NaN — load-bearing, because level geometry contains degenerate triangles and a NaN
normal poisons a whole lightmap.

`reflect(dir, norm)` mirrors a direction across a plane; `slide(dir, norm)` removes only
the component along the normal, leaving the tangential part. Both require a unit normal
and neither renormalizes the result — `slide` deliberately, since the shortened vector *is*
the answer for a character sliding along a wall.

`from_bary` and `from_bary4` reconstruct a point from barycentric weights over three or
four reference points. The weights are not checked and not normalized; the caller owns
that.

## 3-vector — heading and pitch

**Contract** — `setHP(h, p)` writes the unit direction for a heading/pitch pair;
`getHP`, `getH` and `getP` recover them. Together they define what "heading" means for
every direction in the game.

```text
FUNCTION direction_from_heading_pitch(h, p) -> vector
  # heading is measured about +Y starting from +Z and increasing toward -X;
  # pitch lifts out of the horizontal plane toward +Y
  RETURN ( -cos(p)*sin(h), sin(p), cos(p)*cos(h) )

FUNCTION heading_of(v) -> real
  IF v.x and v.z are both effectively zero
    RETURN 0                            # straight up or down: heading is undefined,
                                        # and zero is the agreed answer
  IF v.z is effectively zero
    RETURN IF v.x > 0 THEN -PI/2 ELSE +PI/2
  IF v.z < 0
    RETURN PI - atan(v.x / v.z)         # the far half-turn the arctangent lost
  RETURN -atan(v.x / v.z)

FUNCTION pitch_of(v) -> real
  ground = sqrt(v.x*v.x + v.z*v.z)
  IF ground is effectively zero
    RETURN IF v.y > 0 THEN +PI/2 ELSE -PI/2
  RETURN atan(v.y / ground)
```

**Notes** — `getHP` computes both at once and the two single-value forms repeat its
branches; the duplication is an inlining decision, not a semantic one. The degenerate
answers (heading zero when pointing straight up, pitch ±quarter-turn when the ground
projection vanishes) are contract, not accident: callers rely on getting a usable pair
rather than a signal.

## 3-vector — random directions and points

**Contract** — `random_dir(R)` draws a direction from a caller-supplied generator;
`random_dir(axis, cone_angle, R)` draws one near a given axis; `random_point(box, R)` draws
a point uniformly inside an axis-aligned box centred on the origin; `random_point(r, R)`
draws a direction and scales it by a uniform radius.

```text
FUNCTION random_dir(R) -> vector
  # the vertical component is the cosine of a uniform polar angle, NOT a uniform
  # value in [-1,1] -- so the distribution is denser near the poles than a
  # uniform-on-the-sphere draw would be
  z = cos(R.uniform(0, PI))
  a = R.uniform(0, TWO_PI)
  r = sqrt(1 - z*z)
  RETURN (r*cos(a), r*sin(a), z)
```

**Notes** — The uniform-on-the-sphere line sits in the source, commented out, directly
above the one that ships. A rebuild must keep the shipped, biased one: particle sprays and
creature wander directions were tuned against it and a rebuild that "fixes" the
distribution changes how every emitter in the game looks.

`random_dir(axis, cone_angle, R)` adds a full random direction, scaled by a uniform draw up
to the tangent of the cone angle, to the axis and renormalizes. The scatter it produces is
only approximately a cone of that half-angle — the perturbation is not constrained to the
plane perpendicular to the axis — and `random_point(r, R)` inherits the same bias plus a
radius that is uniform in *r* rather than in volume, so points cluster toward the centre.
All three are tuning inputs rather than statistical primitives.

## 3-vector — orthonormal basis construction

**Contract** — Given a direction, produce the two vectors completing a frame.
`generate_orthonormal_basis(dir, up, right)` treats `dir` as already unit and does not
touch it; `generate_orthonormal_basis_normalized(dir, up, right)` normalizes `dir` first
and takes it by mutable reference for that reason.

```text
FUNCTION basis_from_direction(dir) -> (up, right)
  # choose which component to zero by picking the pair with the larger magnitude,
  # so the vector we build the perpendicular from can never be near zero
  IF abs(dir.x) >= abs(dir.y)
    inv = 1 / sqrt(dir.x*dir.x + dir.z*dir.z)
    up = (-dir.z*inv, 0, dir.x*inv)
  ELSE
    inv = 1 / sqrt(dir.y*dir.y + dir.z*dir.z)
    up = (0, dir.z*inv, -dir.y*inv)
  right = cross(up, dir)
  RETURN (up, right)
```

**Notes** — The cross is taken as `up × dir` rather than `dir × up`; the source marks the
swap with a comment because it is a deliberate departure from the reference this was copied
from, and it fixes the handedness of the resulting frame. Getting it the other way round
mirrors every billboard and every decal built on this basis.

The normalized variant special-cases a direction close to straight up, because the
otherwise-chosen reference axis would be parallel to it. Its threshold is the general-purpose
comparison epsilon rather than something derived from the geometry — coarse, but the
consequence of triggering it slightly early is only a different (still valid) choice of
frame.

## Notes on the file as a whole

The vector bodies are written once over an unspecified component type and then instantiated
for exactly one: single-precision float. The integer 3-vector that shares the declaration
gets only the operations the header inlines. That is the real content of the instantiation
list at the end of the file — a rebuild that writes the vector concretely in float loses
nothing, and a rebuild that writes it generically must remember that only the float form
is ever asked for these operations.
