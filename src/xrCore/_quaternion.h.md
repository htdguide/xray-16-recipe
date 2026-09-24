# src/xrCore/_quaternion.h

> The quaternion: four reals holding a rotation, with the spherical interpolation that every blended animation in the engine runs through.

**Needs** — [`_vector3d.h`](_vector3d.h.md) · [`_matrix.h`](_matrix.h.md) · [`math_constants.h`](math_constants.h.md) · [`xrDebug.h`](xrDebug.h.md) · [`../utils/xrMiscMath/quaternion.cpp`](../utils/xrMiscMath/quaternion.cpp.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md)

**Used by** — [`KinematicsAddBoneTransform.hpp`](../Layers/xrRender/KinematicsAddBoneTransform.hpp.md) · [`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md) · [`quaternion.cpp`](../utils/xrMiscMath/quaternion.cpp.md) · [`SkeletonMotions.hpp`](Animation/SkeletonMotions.hpp.md) · [`_matrix.h`](_matrix.h.md) · [`vector.h`](vector.h.md) · [`PHNetState.h`](../xrServerEntities/PHNetState.h.md)

**Tier floor** — T1: four contiguous reals in a fixed order, stored in shipped animation banks as a memory image and blended per bone per frame.

## Purpose

A rotation stored as a matrix cannot be interpolated — the halfway point between two rotation
matrices is not a rotation. A rotation stored as three angles interpolates badly and locks at
the poles. A quaternion does neither, and that is the only reason this type exists: the
animation layer blends several motions per bone per frame, and the blend must stay a rotation
at every intermediate value.

Everything here is inline except the conversion *from* a transform, which is in
[`quaternion.cpp`](../utils/xrMiscMath/quaternion.cpp.md); the conversion *to* a transform is
in [`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md). The two halves of one operation live in
two files, which is a split by type ownership rather than by anything about the algorithm.

## State

```text
RECORD Quaternion
  x : real       # the imaginary part, i
  y : real       #                     j
  z : real       #                     k
  w : real       # the real part
```

**Invariants**

- **The storage order is x, y, z, w — the real part is last — while every constructor and
  accessor takes its arguments as w, x, y, z, with the real part first.** Nothing warns about
  it. This is the most likely place for a rebuild to go wrong, because the animation banks on
  disk carry the storage order and the code reads in the argument order.
- A rotation is represented by a **unit** quaternion: for a rotation of an angle about a unit
  axis, the real part is the cosine of half the angle and the imaginary part is the axis
  scaled by the sine of half the angle. Nothing enforces unit length; several operations
  produce non-unit results and the caller is expected to normalize.
- **A quaternion and its negation are the same rotation.** Every comparison and interpolation
  on this page has to account for that, and each does so differently — see the interpolation
  and comparison entries.
- The identity rotation is real part one, imaginary part zero.

## `mul` — composition

**Contract** — The quaternion product of two quaternions into a third. Does not normalize.
The destination may alias neither operand in the obvious reading, though in practice each
output component is computed from the inputs before any is written. Asserts both operands are
numerically sane.

```text
FUNCTION compose(a, b) -> Quaternion
  # In the scalar-and-vector reading: the real parts multiply and the
  # imaginary parts' dot product is subtracted; the imaginary part is each
  # real part scaling the other's imaginary part, plus their cross product.
  w = a.w*b.w - a.x*b.x - a.y*b.y - a.z*b.z
  x = a.w*b.x + a.x*b.w + a.y*b.z - a.z*b.y
  y = a.w*b.y - a.x*b.z + a.y*b.w + a.z*b.x
  z = a.w*b.z + a.x*b.y - a.y*b.x + a.z*b.w
```

**Invariants** — The product is **not commutative**, and the convention here is the
mathematical one: `compose(a, b)` applies b first and then a, the same sense as the 4×4
transform's composition. The cross-product term's three signs are the whole content of that
convention and reversing any one of them produces a mirrored world.

**Notes** — The file's own header comment gives a sign for the third imaginary term that
disagrees with what the code computes. The code is the definition; the comment is wrong. A
rebuild reading the original must take the arithmetic, not the prose.

## `add` and `sub` — componentwise

**Contract** — Componentwise addition and subtraction, in place and into a destination. The
results are not rotations — adding two unit quaternions gives a non-unit one — and are used
only as intermediate steps by code that normalizes afterwards.

## `normalize`

**Contract** — Scales to unit length. **Leaves the quaternion untouched** when its length is
below a tiny threshold rather than producing infinities, and reports nothing about having
done so. Allocation-free.

**Invariants** — The threshold is one part in a hundred thousand of unit length. Below it the
quaternion carries no usable direction and the silent no-op is the least harmful behaviour —
but it means a caller cannot distinguish "normalized" from "was degenerate", and a
degenerate quaternion propagates through the animation blend as a near-zero rotation rather
than as a detectable fault.

**Notes** — The magnitude accessor on this type returns the **squared** length, not the
length, which is why normalization takes a square root of it. The naming is a trap and a
rebuild should name the squared form as such.

## `isUnit` and `isValid`

**Contract** — Two cheap sanity predicates.

- **isUnit** — whether the squared length is within one part in a thousand of one. A
  deliberately loose tolerance: a quaternion drifts a little through repeated blending and
  the predicate exists to catch gross error, not drift.
- **isValid** — whether any component squared is negative, which is to say whether any
  component is a non-number. It is a not-a-number test written as arithmetic, because a
  comparison with a not-a-number is false in every direction and this is the shape that
  survives an optimizer that assumes finiteness.

**Notes** — `isUnit` compares the *squared* length against one, which is correct only because
one squared is one. It is right by luck, not by construction, and a rebuild that changes the
target length must remember to square it.

## `inverse` — the conjugate

**Contract** — Four shapes: negate the imaginary part, in place or from another; negate all
four components, in place or from another.

**Invariants** — For a **unit** quaternion the conjugate is the inverse rotation, which is
why negating the imaginary part is called inversion here. For a non-unit quaternion it is
not; nothing checks.

Negating all four components yields the *same* rotation, not the inverse — it is the other
representative of the same rotation on the double cover. The two operations are named almost
identically and mean entirely different things. A rebuild should call the second one
"the antipodal representative" and should have exactly one caller for it: the path that picks
the shorter arc before an interpolation.

## `rotation(axis, angle)` and `get_axis_angle`

**Contract** — Build a rotation from a unit axis and an angle, and recover them. The
construction does **not** normalize the axis, so a non-unit axis produces a non-unit
quaternion. The recovery reports whether an axis exists at all — the identity rotation has no
axis, and in that case the axis is zeroed and the angle set to zero.

```text
FUNCTION from_axis_angle(axis, angle) -> Quaternion
  w = cos(angle / 2)
  s = sin(angle / 2)
  imaginary = axis * s

FUNCTION to_axis_angle(q) -> optional<(axis, angle)>
  s = length of the imaginary part
  IF s <= tight_epsilon THEN RETURN none          # identity: no axis
  axis  = imaginary part / s
  angle = 2 * atan2(s, q.w)
  RETURN (axis, angle)
```

**Invariants** — The recovery uses a two-argument arctangent of the imaginary length against
the real part rather than an arccosine of the real part. That is what makes it correct for
the whole range of angles including those past half a turn, where the real part goes negative
and an arccosine would need a quadrant fix. A rebuild should use the same form.

## `rotationYawPitchRoll` — Euler angles to quaternion

**Contract** — Builds a unit rotation from three angles. Total, allocation-free.

**Invariants** — The composition order matters and must match the transform type's
[rotation sequence](_matrix.h.md#the-conventions), which is bank, then pitch, then heading.
Every authored orientation in the game data goes through one of the two constructions and
they must agree.

**Notes** — The argument names here are yaw, pitch and roll while the transform type calls the
same three heading, pitch and bank, and the script layer calls them x, y and z. Three
vocabularies for one triple, and only the *order* is common to all three. A rebuild should
pick one set of names and convert at the data boundary.

## `slerp` — spherical linear interpolation

**Contract** — Interpolates between two rotations along the shortest arc, writing the result
into this quaternion. The blend parameter runs zero (all of the first) to one (all of the
second), and is asserted to be in that range in debug builds — out of range is a fatal error,
not a clamp. The result is unit when both inputs are. Allocation-free, no branches other than
the ones below. This is the operation the animation blend spends its time in.

```text
FUNCTION slerp(a, b, t) -> Quaternion
  cosine = dot(a, b)            # all four components

  # A quaternion and its negation are the same rotation, so a negative dot
  # means the two representatives are on opposite halves of the sphere and
  # the direct arc is the LONG way round. Flip one of them.
  IF cosine < 0 THEN cosine = -cosine ; flip = -1 ELSE flip = +1

  IF 1 - cosine > loose_epsilon THEN
    angle = arccos(cosine)
    scale_a = sin(angle - t*angle) / sin(angle)
    scale_b = sin(t*angle) / sin(angle)
  ELSE
    # Nearly parallel: the divisions above lose all their precision, and
    # linear interpolation is indistinguishable at this separation anyway.
    scale_a = 1 - t
    scale_b = t

  RETURN a * scale_a + b * (scale_b * flip)
```

**Invariants**

- The sign flip is applied to the *second* quaternion's weight, not to the quaternion itself,
  so neither input is modified. It is what makes the interpolation take the shorter of the
  two arcs, and it is why a naive componentwise blend of animation poses jitters where this
  does not.
- The degenerate branch is on near-parallel inputs, not near-antiparallel ones. After the
  sign flip, antiparallel inputs have become parallel-ish, so the only remaining
  ill-conditioned case is the one handled.
- The result is **not** renormalized. It is unit to within the arithmetic's precision when
  the inputs are unit, and the animation layer renormalizes after accumulating several
  blends rather than after each.

**Notes** — The arccosine used here is **not** the library's. It is a private polynomial
approximation: a degree-seven odd polynomial in the argument approximating arcsine, subtracted
from a quarter turn. That is a deliberate speed-for-accuracy trade made because this runs per
bone per blend per frame — several thousand times a frame with a hundred animated characters
on screen.

The four polynomial coefficients are the one thing on this page that cannot be recovered from
the source: they are a fitted minimax approximation and the fitting criterion is not recorded.
What a rebuild needs to know is the *shape* of the decision — a fast approximate arccosine is
acceptable here because the error feeds an interpolation weight, and a weight that is wrong in
the fourth decimal produces a pose that is wrong by less than the animation's own quantization.
A rebuild should either fit its own polynomial to its own precision or use the library
function and measure whether it matters; copying these four numbers into a different precision
buys nothing.

The approximation is also *not* accurate near the ends of its range, which is the second
reason the near-parallel branch exists.

## `cmp` — rotation equality

**Contract** — Whether two quaternions represent the same rotation within a tolerance,
checking **both** representatives: componentwise equal, or componentwise equal after negating
one. Default tolerance one part in ten thousand.

**Notes** — This is the right way to compare rotations and is worth lifting into a rebuild
unchanged. A componentwise comparison that omits the negated case reports two identical
rotations as different half the time, which is a bug that shows up as animation keys being
duplicated rather than deduplicated.

## `ln` and `exp` — the logarithm and exponential

**Contract** — The quaternion logarithm and its inverse. The logarithm of a unit quaternion
has a zero real part and an imaginary part equal to the rotation axis scaled by half the
angle; the exponential takes that back. Both guard the degenerate case where the imaginary
part is near zero by producing a zero imaginary result rather than dividing by it.

**Notes** — These exist to support interpolation in the logarithmic domain — a linear blend of
logarithms, exponentiated back, is a legitimate alternative to spherical interpolation and
extends naturally to blending more than two rotations, which spherical interpolation does not.
The file's own commentary argues for that approach. Whether the engine's animation blend
actually takes it is a question for the animation chapter; what matters here is that the two
operations are a matched pair and are meaningful only on unit quaternions.

## `set(transform)` — from a rotation matrix

**Contract** — Declared here, implemented in
[`quaternion.cpp`](../utils/xrMiscMath/quaternion.cpp.md). Note the failure mode documented
there: there is one input for which the destination is left **completely unwritten**.

## The tolerance constants

Five named tolerances are declared at the top of the file and undeclared at the bottom, so
they do not escape:

| Name | Value | Used for |
|---|---|---|
| unit tolerance | one part in a thousand | how far from unit length still counts as a rotation |
| zero tolerance | one part in a hundred thousand | below this, normalization declines to act |
| trace tolerance | one tenth | the matrix-to-quaternion branch threshold |
| axis-angle tolerance | one part in ten thousand | declared, unused |
| general epsilon | one part in a hundred thousand | declared, unused |

**Notes** — Two of the five are dead. The trace tolerance is consumed in
[`quaternion.cpp`](../utils/xrMiscMath/quaternion.cpp.md) rather than here. None of the five
is derived from anything stated; they are the tolerances of a single-precision implementation
and a rebuild working in a different precision must re-derive rather than copy them.

## Validity

**Contract** — All four components finite and normal. Distinct from `isValid`, which only
rejects non-numbers; this one also rejects infinities and subnormals.
