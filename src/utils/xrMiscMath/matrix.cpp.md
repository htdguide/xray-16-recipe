# src/utils/xrMiscMath/matrix.cpp

> The out-of-line body of the 4×4 transform: composition, inversion, the rotation
> constructors, and the Euler-angle conversions.

**Needs** — [`xrCore/_matrix.h`](../../xrCore/_matrix.h.md) · [`xrCore/_quaternion.h`](../../xrCore/_quaternion.h.md) · [`xrCore/_vector3d.h`](../../xrCore/_vector3d.h.md) · [`xrCommon/math_funcs_inline.h`](../../xrCommon/math_funcs_inline.h.md) · [`xrCore/xrDebug.h`](../../xrCore/xrDebug.h.md) · [Platform assumptions](../../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)
**Used by** — [`_matrix.h`](../../xrCore/_matrix.h.md)
**Tier floor** — T2: the arithmetic is pure, but the sixteen floats must be contiguous and in this exact order because the same storage is handed to the graphics device as a constant buffer, and the routines run thousands of times per frame

## Purpose

Every transform in the engine — bone poses, object placements, the view and projection
matrices, particle frames — is one 4×4 record, and this file holds every operation on it
that is too long to inline. The conventions are stated once in
[the directory README](README.md#transform-conventions) and are assumed here: row-vector
multiplication, translation in the fourth row, rotation sequence Z then X then Y, and a
positive angle turning clockwise when looking along the axis.

The split from the header is arbitrary in the sense that nothing forces it — these are the
routines somebody decided were too big to inline. It is *not* arbitrary that they live in
this module rather than in the core module: see
[the README's note on why this module exists](README.md#why-this-is-a-separate-module).

## State

`Stateless.` Every routine writes into a caller-owned transform record and returns it.

## `identity`

**Contract** — Writes the identity transform. Total, allocation-free.

## `mul(A, B)` — composition

**Contract** — Writes the composition of two transforms into a *third*, distinct record.
The destination may not alias either operand; the engine asserts this rather than
defending against it, because the in-place forms (`mulA_44`, `mulB_44` in the header) exist
precisely to copy an operand out of the way first. Total.

**Invariants** — Argument order is reversed from the underlying matrix product. `mul(A, B)`
composes so that **B is applied first and A second**. This is the single most
error-prone line in the module: a rebuild that reads the arguments in the other order
produces a world that is subtly, consistently wrong in a way no assertion catches.

```text
FUNCTION compose(a, b) -> transform
  # result = b × a in the row-major product, so that a point p
  # transformed by the result equals (p through b) through a
  FOR EACH row i IN 0..3
    FOR EACH col j IN 0..3
      result.m[i][j] = SUM over k IN 0..3 OF b.m[i][k] * a.m[k][j]
  RETURN result
```

## `mul_43(A, B)` — affine composition

**Contract** — The same composition, restricted to transforms whose fourth column is
`(0, 0, 0, 1)` — that is, every transform that is not a projection. Saves a quarter of the
work by not computing the projection column and writing its known constants instead.

**Invariants** — Both operands must actually be affine. Feeding a projection matrix in
silently discards the projective terms; nothing checks.

**Notes** — This is the form used for the per-bone and per-object transforms, which is
almost all of the composition the engine does in a frame, so the saving is real rather
than cosmetic.

## `invert(a)` and `invert_b(a)` — affine inverse

**Contract** — Inverts the affine part of a transform: the 3×3 linear block by cofactors
over its determinant, then the translation as the negated original translation run through
that inverse. The fourth column is forced back to `(0, 0, 0, 1)`. `invert` asserts an
invertible linear block; `invert_b` reports failure to the caller instead, for the paths
that can legitimately be handed a degenerate frame (a collapsed bone, a zero-scaled
object). Destination and source must be distinct.

**Invariants** — This is a *general* linear inverse, not a transpose, so it is correct for
transforms carrying scale and shear. It is wrong for any transform with a projection
column; use the full inverse for those.

```text
FUNCTION invert_affine(a) -> optional<transform>
  det = a.m11*(a.m22*a.m33 - a.m23*a.m32)
      - a.m12*(a.m21*a.m33 - a.m23*a.m31)
      + a.m13*(a.m21*a.m32 - a.m22*a.m31)
  IF abs(det) <= smallest_positive_real
    RETURN none                          # invert() asserts here instead
  inv_det = 1 / det
  linear_part = adjugate(a.linear_part) scaled by inv_det
  # the translation of the inverse is the original translation, negated,
  # expressed in the inverted frame
  result.translation = -(a.translation through linear_part)
  result.projection_column = (0, 0, 0, 1)
  RETURN result
```

## `invert_44(a)` — full inverse

**Contract** — The general 4×4 inverse by cofactor expansion, for transforms that carry a
projection column — the view-projection matrix and anything derived from it. Asserts a
non-zero determinant. Destination and source must be distinct.

**Notes** — Written as an unrolled cofactor expansion with the six 2×2 minors of the
bottom two rows hoisted and shared. That sharing is the only reason the expansion is worth
writing out rather than looping; a rebuild that loops is correct but slower, and this
routine is called a handful of times per frame, so looping is a defensible choice here in
a way it is not for `mul`.

## `transpose(source)`

**Contract** — Full 4×4 transpose into a distinct destination. Used to hand matrices to a
graphics interface with the opposite row/column convention, and to build normal matrices.

## `rotateX`, `rotateY`, `rotateZ`

**Contract** — Each writes a pure rotation about one world axis, zero translation. Total.

**Invariants** — A positive angle rotates clockwise when looking *along* the axis
direction. This is the opposite of the mathematical convention most readers carry, and it
is baked into every authored angle in the game data, so it cannot be flipped in a rebuild.

## `rotation(dir, norm)` — frame from a direction and an up vector

**Contract** — Builds a rotation whose third basis row is `dir`, second is `norm`, and
first is the normalized cross product of `norm` with `dir`. Zero translation.

**Invariants** — `dir` and `norm` must already be unit length and roughly perpendicular;
only the computed right vector is normalized. Feeding a non-unit pair produces a
non-orthonormal frame that will not survive inversion by the affine inverse above.

## `rotation(axis, angle)` — rotation about an arbitrary axis

**Contract** — The axis-angle rotation. Zero translation. The axis is assumed unit and is
**not** normalized here; a non-unit axis silently scales as well as rotates.

```text
FUNCTION rotation_about(axis, angle) -> transform
  c = cos(angle)
  s = sin(angle)
  # each diagonal entry keeps the axis component and rotates the remainder;
  # each off-diagonal pair is the symmetric (1-c) term plus the antisymmetric sin term
  FOR EACH pair of axes (u, v) with third axis w
    diagonal[u] = axis.u*axis.u + (1 - axis.u*axis.u)*c
    entry[u][v]  = axis.u*axis.v*(1 - c) + axis.w*s
    entry[v][u]  = axis.u*axis.v*(1 - c) - axis.w*s
  translation = zero
```

## `mapXYZ` … `mapZYX` — the six axis permutations

**Contract** — Six constant transforms, one per permutation of the three basis rows. Used
to convert data authored in another tool's coordinate handedness into the engine's.

**Notes** — Three of the six are odd permutations and therefore *reflections*: they flip
handedness, which reverses triangle winding and negates every cross product downstream. A
rebuild must keep all six — the engine picks one by name from data, and dropping the
reflecting ones silently changes which imported assets come in mirrored.

## `mul(A, scalar)`, `mul(scalar)`, `div(A, scalar)`, `div(scalar)`

**Contract** — Elementwise scale of all sixteen entries, in place or into a destination.
The division forms assert that the divisor's magnitude exceeds one part in a million and
then multiply by the reciprocal, so a rebuild gets one division rather than sixteen.

**Notes** — Scaling all sixteen entries including the fourth row and column is meaningful
only for matrices being used as generic numeric arrays (interpolation of pose deltas,
accumulating weighted transforms); applied to a transform it scales the translation and
the homogeneous divisor together and is not a spatial scale.

## `setHPB(h, p, b)` — Euler angles to transform

**Contract** — Builds a pure rotation from heading, pitch and bank, composed in the order Z
(bank) then X (pitch) then Y (heading). Zero translation. Total.

**Invariants** — The rotation sequence is fixed and load-bearing: every authored orientation
in the game data, and every orientation the script layer sets, is three numbers in this
order. Changing the sequence changes where every scripted object faces.

## `getHPB(h, p, b)` — transform to Euler angles

**Contract** — Recovers heading, pitch and bank from the rotation part, inverting
`setHPB`. Total, but lossy at the poles.

```text
FUNCTION to_euler(t) -> (heading, pitch, bank)
  # cy measures how much of the frame survives projection out of the pitch axis;
  # near zero the heading and bank rotations act about the same axis and are
  # no longer separable (gimbal lock)
  cy = sqrt(t.row2.y * t.row2.y + t.row1.y * t.row1.y)
  IF cy > gimbal_threshold
    heading = -atan2(t.row3.x, t.row3.z)
    pitch   = -atan2(-t.row3.y, cy)
    bank    = -atan2(t.row1.y, t.row2.y)
  ELSE
    # give the whole rotation to heading and set bank to zero,
    # which reproduces the same frame even though it is not the same triple
    heading = -atan2(-t.row1.z, t.row1.x)
    pitch   = -atan2(-t.row3.y, cy)
    bank    = 0
  RETURN (heading, pitch, bank)
```

**Notes** — The gimbal threshold is sixteen times the smallest representable step at one.
The factor of sixteen is not derived from anything; it is "a few ulps of slack". What
matters for a rebuild is that the threshold be small, non-zero, and scaled to the working
precision — a fixed decimal constant chosen for single precision becomes far too coarse if
the rebuild works in double.

## `rotation(quaternion)` and `mk_xform(quaternion, translation)`

**Contract** — Convert a unit quaternion to a rotation transform. `mk_xform` additionally
writes the translation row; the rotation half is identical between the two. Total, no
normalization of the input.

**Invariants** — The quaternion must be unit. The conversion is the standard one and it
scales by the squared norm, so a quaternion that has drifted off the unit sphere yields a
transform with baked-in scale — which is why the animation layer renormalizes after
blending rather than here.

**Notes** — Two copies of the same nine expressions exist because one writes a zero
translation and the other writes a supplied one. That duplication is an artifact; a
rebuild writes the rotation once and sets the translation separately.
