# src/xrGame/ik/math3d.cpp

> The solver's linear algebra, in the solver's own conventions — row vectors multiplying
> on the left, quaternions scalar-first, rotations that read transposed against the
> engine's. A rebuild deletes this file and uses its own; what it must not delete is the
> list of conventions, because every formula in the directory assumes them.

**Needs** — [`math3d.h`](math3d.h.md)
**Used by** — reached through its declarations in [`math3d.h`](math3d.h.md); callers name that, not this file.
**Tier floor** — T2. Fixed-size float arithmetic with no allocation, called a few hundred
times per frame. Nothing here needs explicit layout — the one place that looks like it
does, reinterpreting the engine's transform as this one's because both are sixteen
contiguous floats, is a bug a rebuild should convert away rather than reproduce.

## Purpose

This file is entirely **incidental in substance and load-bearing in convention**. Every
operation in it exists somewhere in the engine's own math layer; none of them is
interesting. What is load-bearing is the handful of choices that make the two layers
different, because the formulas in
[`Dof7control.cpp`](Dof7control.cpp.md), [`eulersolver.cxx`](eulersolver.cxx.md) and
[`limb.cxx`](limb.cxx.md) were derived under *these* conventions and silently produce
mirrored poses under any other.

## State

```text
CONSTANT identity : Transform    # the 4x4 identity, read by the principal-axis
                                 # rotation constructor so it need only write four cells
```

## Conventions — the only part of this file that must survive

**Row vectors.** A point is a row and transforms multiply on its right: `p' = p · M`. A
chain composes left to right in the order the bones are traversed. Column-vector code
reads every product here backwards.

**Row-major storage, translation in the last row.** The three basis vectors are rows 0–2;
the position is row 3. A column-major library's memory image of the same transform is the
transpose, which is exactly the trap the engine's own type falls into here.

**Rotations about a principal axis are built transposed relative to the common
convention.** A rotation about *z* by θ places `+sin θ` at row 0 column 1 and `−sin θ` at
row 1 column 0 — the transpose of the textbook column-vector form, which is the correct
partner of the row-vector rule above. Get this wrong and every joint rotates the wrong
way while still producing a valid-looking pose.

**Quaternions are scalar-first**, `(w, x, y, z)`, and the matrix they build follows the
same row-vector convention.

**The axis recovered from a rotation matrix is always paired with an angle in the first
half turn.** Axis sign carries the direction; the angle is never negative. Near identity
and near a half turn the axis is not determined, and the code substitutes the *z* axis
with a zero angle rather than returning a degenerate answer.

## `hmatmult` · `rmatmult` · `matmult`

**Contract** — three multiplies of decreasing generality: arbitrary 4×4, rigid transform
(rotation plus translation, no scale or perspective), and rotation only. Each accepts an
output that aliases either input. Each is a fixed number of multiplies with no branching.

**Notes** — the three exist because the general case costs sixty-four multiplies and the
rigid case costs thirty-six for a result that is identical whenever the inputs really are
rigid. The rotation-only case is used when a translation is known to be meaningless and
would only accumulate rounding.

The *decision* is that the solver never needs a general 4×4 — every matrix crossing its
boundary is a rigid transform. That is asserted informally: the rigid multiply produces
garbage on a matrix with scale, silently. A rebuild with one multiply and a compiler that
can see the constant zeroes loses nothing.

## `inverthomomatrix` · `invertrmatrix`

**Contract** — inverse of a rigid transform, and of a pure rotation. Both by transpose:
the rotation part is transposed, and the rigid version's translation becomes the negated
dot of the original translation with each basis row. No general inverse exists in this
file and none is needed.

**Invariants** — the input must be orthonormal. Nothing checks it; a non-rigid input gives
a wrong answer rather than an error.

## `vecmult` · `vecmult0`

**Contract** — apply a transform to a point, and to a direction. They differ only in
whether the translation row is added. Both are safe when the output aliases the input.

**Notes** — this is the pair that makes the point/direction distinction explicit instead
of relying on a fourth component. It is the right decision and a rebuild should keep the
distinction whatever its representation: the ground normal and the hip-to-foot direction
must not pick up translations.

## `rotation_axis_to_matrix` · `axisangletomatrix`

**Contract** — build a rotation from a unit axis and an angle. Two entry points with the
same meaning: the second special-cases an axis that lies exactly along *x*, *y* or *z* and
writes the small form directly, the first always uses the general formula. Neither
normalizes the axis — a non-unit axis silently produces a matrix that also scales.

**Notes** — the special-casing is a speed decision from a period when the general form's
nine multiplies mattered. The *reason it is still correct* is subtler and worth keeping:
the principal-axis branches also handle a negative axis by flipping the sine, so
`(0,0,−1)` and angle θ means what `(0,0,1)` and −θ means. A rebuild that keeps only the
general form gets this for free.

## `rotation_principal_axis_to_matrix` · `rotation_principal_axis_to_deriv_matrix`

**Contract** — a rotation about *x*, *y* or *z* selected by name, and **the derivative of
that rotation with respect to its angle**. The derivative has no rotation-matrix
properties — it is not orthonormal and its determinant is not one. It exists so that the
rate of change of a joint angle can be propagated through a chain without finite
differences.

**Notes** — the derivative entry point is declared and built but the shipped solve path
never calls it. It belongs to the joint-limit machinery, which is switched off — see
[`jtlimits.cxx`](jtlimits.cxx.md). A rebuild should implement it last.

The selector is a character, and an unrecognized one silently means *z*. That is a
defaulting bug rather than a decision; a rebuild should use a closed enumeration.

## `rotation_matrix_to_axis`

**Contract** — recover the axis and angle of a rotation matrix. The angle comes from the
trace; the axis from the antisymmetric part, normalized. The angle is always in the first
half turn.

```text
FUNCTION rotation_to_axis_angle(R) -> (axis, angle)
  angle <- arccos( (R.trace3 - 1) / 2 )

  # Near identity the axis is undefined; near a half turn the antisymmetric part
  # vanishes and the axis cannot be recovered this way at all. Both are answered with
  # a zero rotation about z rather than a NaN. The half-turn case is a genuine
  # approximation, not just a convention: a rebuild that needs it correct must
  # recover the axis from the symmetric part instead.
  IF angle is near 0 OR near half_turn
    RETURN ((0,0,1), 0)

  axis <- ( R[1][2] - R[2][1],
            R[2][0] - R[0][2],
            R[0][1] - R[1][0] )
  normalize axis
  RETURN (axis, angle)
```

## `matrixtoq` · `qtomatrix` · `axistoq` · `qtoaxis`

**Contract** — the four conversions between a rotation matrix, a quaternion and an
axis-angle pair. The matrix-to-quaternion direction is the interesting one: it selects
among four expressions by which of the diagonal-derived quantities is safely positive,
falling through scalar-part, then *x*, then *y*, then a fixed *z*-only answer, so that the
division is never by a small number. The result is normalized before it is returned.

**Notes** — the fall-through ladder is the standard Shepperd selection written as nested
conditionals; the *decision* is that the largest component is found by testing rather than
comparing, and that the final rung is a constant rather than an error.

## `linterpmatrix` · `vecinterp`

**Contract** — interpolate between two rigid transforms, and between two vectors. The
rotation is interpolated by taking the relative rotation, converting to axis-angle,
scaling the angle, and re-applying — a spherical interpolation along the shortest arc. The
translation is interpolated linearly. At parameter zero the result is the first input, at
one the second.

**Notes** — nothing in the shipped solve path calls this. The blend the feature actually
uses is a rate limit on the goal, not an interpolation between poses — see
[`IKLimb.cpp`](IKLimb.cpp.md). A rebuild need not port it.

## `project` · `project_plane` · `angle_between_vectors`

**Contract** — project a vector onto another; project a vector onto the plane with a given
normal; and the **signed** angle from one vector to another measured about a given axis,
in a full turn's range. The angle function projects both inputs onto the plane first, so
it is well defined for vectors that are not perpendicular to the axis.

**Notes** — this trio is the swivel angle's arithmetic. `angle_between_vectors` is how a
knee position on its circle becomes the scalar ψ that parameterizes the whole solve; the
axis it measures about is the circle's normal, which is the hip-to-foot direction. Getting
its sign convention wrong mirrors the knee.

An earlier implementation using a cross product and an explicit parallel-vector test is
present and disabled. The shipped one — project both, cross, then a quadrant-correct
arctangent of the cross against the dot — needs no special case at 0 or a half turn, which
is why it replaced the other.

## `find_normal_vector`

**Contract** — given a vector, produce *some* unit vector perpendicular to it. Any will
do. A zero input yields a zero output.

```text
FUNCTION any_perpendicular(v) -> vector
  # Pick the component of smallest magnitude and build the perpendicular in the plane
  # of the other two. Choosing the smallest keeps the result far from degenerate: the
  # two components it is built from are the two largest, so it can never be a
  # near-zero vector that normalization amplifies into noise.
  count how many components are below 1e-8
  MATCH count
    3 -> RETURN zero                       # no perpendicular to the zero vector
    2 -> RETURN the unit axis of the single small component
    otherwise ->
      i <- index of the smallest-magnitude component
      build the vector that swaps and negates the OTHER two components, zero at i
      normalize it
```

**Notes** — the "pick the smallest component" rule is the one genuinely non-obvious thing
in this file and it is worth stating in any rebuild: the naive "cross with the *z* axis"
fails exactly when the input is near *z*, which for a leg pointing straight down is the
common case, not the rare one.

## `norm` · `unitize` · `unitize4` · `get_translation` · `set_translation`

**Contract** — vector length; normalize in place returning the prior length; the same for
a four-component vector; and read or write a transform's translation. The normalizers
leave a zero vector untouched and return zero rather than dividing.

**Notes** — the translation reader has an overload that returns only the magnitude. That
is how both link lengths are measured out of the bind pose at setup time, once, and it is
the only place the chain's geometry enters the solver.
