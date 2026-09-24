# src/xrGame/MathUtils.h

> The game layer's own small math vocabulary: horizontal-plane operations, projections onto axes and planes, ballistic launch solving, and the vector helpers that let engine vectors and the physics library's raw coordinate triples be the same bytes.

**Needs** — [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [`utils/xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — [`MathUtils.cpp`](MathUtils.cpp.md) · [`PHDebug.cpp`](PHDebug.cpp.md)
**Tier floor** — T1: its central premise is that a three-component vector and a bare triple of reals are interchangeable without conversion, which requires a guaranteed memory layout

## Purpose

The engine already has a math layer. This header exists for the things that math layer does
not know about: the game's world is a horizontal plane with a vertical axis, and a great deal
of game logic wants "how far apart horizontally", "which way is he facing on the ground",
"can I throw this there". Those go here.

The second, less visible job is the boundary with the rigid-body library, which talks in bare
triples of reals rather than in the engine's vector type. Most of this header is small
operations on such triples so that the physics-facing code can work with the library's own
storage without copying. A rebuild whose physics binding converts at the boundary deletes
about half of this file.

The header carries all the substance; [`MathUtils.cpp`](MathUtils.cpp.md) contains only a
scratch routine and its benchmark.

## State

`Stateless`, except for one small record:

```text
RECORD InertialValue          # a first-order low-pass filter over a scalar
  value    : real
  inertion : real             # invariant: 0 < inertion < 1, asserted at construction

FUNCTION InertialValue.update(sample)
  value = inertion * value + (1 - inertion) * sample
```

Used wherever a measured quantity must not jump — a camera's lean, a weapon's sway. The
inertion is fixed for the life of the value, deliberately: a filter whose time constant moves
is not a filter.

## Reinterpretation between vectors and coordinate triples

**Contract** — four functions view an engine vector as a triple of reals and back, with no
copy.

**Invariants** — this requires the vector type to be exactly three reals laid out in order,
with no header and no padding. That is the assumption; it is never stated anywhere else and
it is what makes the physics binding cheap.

**Notes** — the problem being solved is that two libraries name the same three numbers
differently. A rebuild with a language that cannot reinterpret memory should instead make its
vector type *be* the physics library's triple, or accept the copy — the copy is three reals
and is not the reason this exists; the reason is that it appears in the physics inner loop.

## `accurate_normalize`

**Contract** — normalizes a vector, in place, without losing precision on a vector whose
components are all tiny or all enormous. Returns a fixed unit vector along the first axis for
a zero vector rather than failing.

```text
FUNCTION accurate_normalize(v)
  # Fast path: the ordinary normalization, good for everything of sane magnitude.
  sq = v.x*v.x + v.y*v.y + v.z*v.z
  IF sq > TINY THEN v = v * reciprocal_sqrt(sq); RETURN

  # Slow path: divide through by the LARGEST component first, so the squares that follow
  # cannot underflow. The largest component's own contribution becomes exactly 1 and is
  # restored afterwards with its original sign.
  find the component with the largest magnitude
  IF that magnitude is zero THEN v = (1, 0, 0); RETURN     # a default, not an error
  divide the other two by it
  l = reciprocal_sqrt(other_a^2 + other_b^2 + 1)
  the two others become their scaled values times l
  the largest becomes l with the original component's sign
```

**Notes** — the threshold is the smallest normal single-precision value's neighbourhood. Below
it, squaring underflows to zero and the ordinary normalization divides by zero. The slow path
costs a divide and a branch and is taken almost never; the decision is to pay for correctness
on a path that would otherwise produce a silent NaN in the physics solver.

Returning a fixed axis for a zero vector rather than reporting failure is the decision every
caller depends on: the callers are orientation code that must produce *some* frame.

## Horizontal-plane operations

**Contract** — `dXZMag`, `dXZDot` and `dXZDotNormalized` are magnitude, dot product and
normalized dot product computed on the two horizontal components only, ignoring height. The
normalized form divides by the product of the two horizontal magnitudes and is undefined for
a vertical vector.

**Notes** — these are the reason the file exists. Almost every "is he facing me", "how far is
that", "turn toward him" question in the game layer is a horizontal question, because
creatures stand on the ground. A rebuild that answers them in three dimensions will find
that a creature on a staircase cannot see a creature below it.

The axis convention is fixed by the whole engine: the second component is up.

## Triple arithmetic

**Contract** — `dVectorSet`, `dVectorSetInvert`, `dVectorSetZero`, `dVectorAdd`,
`dVectorAddMul`, `dVectorSub`, `dVectorInvert`, `dVectorMul` and their four-component
counterparts are componentwise assignment and arithmetic on bare triples and quadruples, in
both in-place and three-operand forms. `dQuaternionSet` is the four-component copy under
another name, because a quaternion is stored as four reals.

**Notes** — nothing here is a decision; it is the arithmetic the physics boundary needs
expressed over the physics library's storage. A rebuild writes it once for its own vector type
and deletes the whole group.

## `dVectorInterpolate`

**Contract** — linear interpolation between two triples. The two-argument form **destroys its
second operand**, which the three-argument form works around by copying first.

**Notes** — the destructive form is a real hazard and the three-argument form exists only to
avoid it. A rebuild should write the non-destructive one and nothing else.

## `dVectorLimit`

**Contract** — clamps a vector's magnitude to a limit, writing the result to a separate output
and returning whether clamping occurred. The return value is the point: callers branch on "was
this clamped" to decide whether a motion request was satisfiable.

## `dVectorDeviation` · `dVectorDeviationAdd` · `dMatrixSmallDeviation` · `dMatrixSmallDeviationAdd`

**Contract** — measures of how far one state is from another, accumulated across several
measurements by the `Add` variants. The vector forms are simple differences. The matrix forms
take the difference of three specific off-diagonal entries of two rotation matrices.

**Invariants** — the three entries chosen are the off-diagonal pairs that, for a *small*
rotation difference, approximate the rotation vector between the two frames. This is only
valid for small differences — hence the name — and produces nonsense for large ones.

**Notes** — `dVectorDeviationAdd` accumulates `from - to` while `dVectorDeviation` computes
`to - from`. The two have opposite sign conventions. That looks like a bug and the recipe
cannot establish which is intended; a rebuild should pick one and check every caller.

The specific matrix indices are into a row-major three-by-four layout with padding, which is
how the physics library stores a rotation. The indices are therefore a property of that
library's storage, not of the mathematics.

## `twoq_2w`

**Contract** — recovers the angular velocity that carries one orientation to another in a given
time. Pure; degenerate when the two orientations are identical, which it handles by skipping
the angular scaling and returning the (then near-zero) axis directly.

```text
FUNCTION twoq_2w(from, to, dt) -> angular velocity
  # The axis comes from the vector part of the relative rotation; the magnitude comes from
  # the angle between the two orientations divided by the interval.
  cosine = from.w * to.w + dot(from.vector, to.vector)
  w      = cross(from.vector, to.vector) + to.vector * from.w - from.vector * to.w
  k      = 2 / dt
  sin_sq = 1 - cosine^2
  IF sin_sq > TINY THEN k = k * arccos(cosine) / sqrt(sin_sq)
  RETURN w * k
```

**Notes** — used to drive a physics body toward an animated pose: the animation produces the
target orientation and this produces the angular velocity to hand the solver. That is the
whole reason it exists and it belongs to the ragdoll-blending story rather than to geometry.

The sign of the cross-product term is marked as uncertain in the original. It is consistent
with the rest of the engine's quaternion convention, but the recipe cannot establish
independently that it is right.

## Projections

**Contract** — three related operations, all in place or into an output:

- `prg_pos_on_axis` — replaces a position with its projection onto a line given by a point and
  a direction. The direction need not be unit; it is divided out twice.
- `prg_pos_on_plane` — moves a position onto a plane given by a normal and an offset, and
  returns the signed distance moved. The normal must be unit.
- `prg_on_normal` — removes a direction's component along a normal, leaving the part parallel
  to the surface. The sign convention adds the projection rather than subtracting it, which
  is correct for an inward-pointing normal.
- `restrict_vector_in_dir` — removes a vector's component along a direction **only when that
  component is positive**. This is the "you may not move into the wall, but you may move away
  from it" rule, and it is the single most-used of the group: it is how the character
  controller resolves a contact without cancelling legitimate motion.

## Ballistics

The throw solver. Given a displacement to cover and a launch speed, find the launch angles.

**Contract** — `TransferenceAndThrowVelToTgA` returns how many solutions exist (zero, one or
two) and their tangents; `TransferenceAndThrowVelToThrowDir` turns those into unit launch
directions. `ThrowMinVelTime` gives the flight time of the minimum-energy throw, and
`TransferenceToThrowVel` converts a displacement and a flight time into the launch velocity
that achieves it.

```text
FUNCTION TransferenceAndThrowVelToTgA(displacement, speed, gravity) -> (count, tangents, horizontal_distance)
  horiz_sq = displacement.x^2 + displacement.z^2
  v_sq     = speed^2
  # The discriminant of the standard ballistic-range quadratic in the launch tangent.
  disc = 1 - gravity / v_sq^2 * (2 * displacement.up * v_sq + gravity * horiz_sq)
  IF disc < 0 THEN RETURN (0, -, -)          # the target is out of range at this speed
  horizontal_distance = sqrt(horiz_sq)
  scale = v_sq / (gravity * horizontal_distance)
  IF disc == 0 THEN RETURN (1, both tangents = scale, horizontal_distance)   # exactly at range
  root = sqrt(disc)
  RETURN (2, [scale*(1 - root), scale*(1 + root)], horizontal_distance)
```

**Invariants** — the two solutions are the flat trajectory and the lobbed one, in that order.
Callers that want "throw it over the wall" take the second; callers that want "throw it fast"
take the first. The ordering is load-bearing.

```text
FUNCTION TransferenceAndThrowVelToThrowDir(displacement, speed, gravity) -> list of directions
  # Both directions share the displacement's horizontal part; only the vertical differs.
  count, tangents, horiz = TransferenceAndThrowVelToTgA(...)
  IF count == 0 THEN RETURN empty
  each direction = displacement with its vertical component replaced by tangent * horiz,
                   then normalized
```

```text
FUNCTION TransferenceToThrowVel(displacement, time, gravity) -> velocity
  # The velocity that covers the displacement in exactly `time` under gravity:
  # horizontally, distance over time; vertically, the same plus the gravity correction.
  v = displacement / time
  v.up = v.up + time * gravity / 2
  RETURN v

FUNCTION ThrowMinVelTime(displacement, gravity) -> real
  RETURN sqrt(2 * magnitude(displacement) / gravity)
```

**Notes** — `ThrowMinVelTime` uses the displacement's *full* magnitude, not its horizontal
part, which makes it an approximation of the minimum-energy flight time rather than an exact
solution. It is used as a starting guess, and that is adequate; a rebuild should not present
it as exact.

## Scalar helpers

**Contract** — `fsignum` (which returns positive one for zero, not zero), `save_max` and
`save_min` (running extrema, written in place), `limit_above` and `limit_below` (one-sided
clamps, in place).

**Notes** — the sign function's treatment of zero is deliberate and depended upon: its callers
want a direction to move in, and zero is not one.

## `DET` · `valid_pos` · rotation-matrix validation

**Contract** — the determinant of the upper three-by-three block of a transform; a very loose
"is this position anywhere sane" predicate that grows a bounding box by a huge margin before
testing containment; and a debug-only check that a matrix is a rotation, by testing its
determinant against one within a wide tolerance.

**Notes** — the tolerance on the rotation check is very wide (it admits a thirty-five percent
scale). It is a "this matrix has become garbage" detector, not a validity check, and the
original's own comment says so. Both of these are diagnostics and neither belongs in a
rebuild's shipping path.

## `check_obb_sise`

**Contract** — true when an oriented box has any non-degenerate extent. Guards code that would
divide by a zero half-size.

## The ordering macros

**Contract** — four code-generating forms that branch on the relative order of three values and
run caller-supplied code for each: the largest, the smallest, the two that are not the
smallest, and a full ordering.

**Notes** — these exist to sort three numbers *and carry along whatever is associated with
them* without writing a swap, in code that was hot enough that the author did not want a
function call or an array. A rebuild should sort a small array of pairs, or index the
components, and measure before worrying. Nothing about the branch structure is load-bearing;
the *outcomes* are — largest, smallest, middle, full order.
