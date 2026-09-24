# src/xrCore/_matrix33.h

> The 3×3 matrix: the rotation-and-inertia workhorse the physics bridge and the box-versus-triangle test speak in, with a Jacobi eigen-decomposition attached.

**Needs** — [`_vector3d.h`](_vector3d.h.md) · [`_matrix.h`](_matrix.h.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)

**Used by** — [`Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [`_obb.h`](_obb.h.md) · [`vector.h`](vector.h.md)

**Tier floor** — T1: nine contiguous reals handed across the boundary to the rigid-body library, which reads them as its own rotation layout.

## Purpose

Three things want a 3×3 matrix rather than the engine's usual 4×4: the rigid-body library,
whose rotations are 3×3 and whose inertia tensors are symmetric 3×3; the oriented bounding
box in [`_obb.h`](_obb.h.md), whose orientation has no translation to carry; and the
separating-axis test between a box and a triangle, which works entirely in a rotation
between two frames. Carrying a fourth row and column through any of those means moving and
multiplying numbers that are known to be zero.

**Read the convention warning below before anything else on this page.** This type does not
use the same multiplication convention as [`_matrix.h`](_matrix.h.md), and one of its
operations does not use the same convention as the rest of the type.

## State

```text
RECORD Matrix3
  row1 : (real, real, real)    # basis vector, addressable as i
  row2 : (real, real, real)    #                              j
  row3 : (real, real, real)    #                              k
```

Nine reals, row by row, no padding. Addressable three ways — by element name, as three named
basis vectors, and as a 3×3 array — over one set of bytes.

**Invariants** — Nothing is enforced. The type is used for rotations (orthonormal), for
inertia tensors (symmetric, positive-definite) and for arbitrary linear maps, and no
operation checks which it has been handed.

## The convention warning

The 4×4 transform multiplies **row vectors on the left**: `p' = p × M`, basis in the rows.
This type's vector operations multiply **column vectors on the right**: `p' = M × p`, basis
in the columns. The two are transposes of each other, which is why the conversion between the
two types is sometimes a copy and sometimes a transpose, and why the transposing product
variants below exist at all.

The exception is `transform_dir`, which follows the 4×4 convention *and* ignores the third
component of its input — see its own entry. A rebuild should pick one convention for the
whole engine and convert exactly once, at the boundary with the rigid-body library.

## `set`, `identity`, `transpose`

**Contract** — Copy from another 3×3 or from the upper-left block of a 4×4; write the
identity; transpose into a destination or in place. All allocation-free.

**Notes** — Taking the upper-left block of a 4×4 is a plain copy, *not* a transpose, despite
the convention mismatch above. So a rotation converted this way comes out transposed — which
is to say, inverted. Every caller that does this is either compensating deliberately or is
using the result in a context where the transposition is what it wanted; the type gives no
help in telling which.

## `set_rapid` — the handedness conversion

**Contract** — Copies the upper-left block of a 4×4 while negating the third row and the
third column, except for the entry where they cross.

```text
FUNCTION flip_third_axis(m) -> Matrix3
  # This is diag(1, 1, -1) applied on both sides: it expresses the same
  # linear map in a frame whose third axis points the other way. Entries
  # touched once change sign; the corner touched twice does not.
  result[a][b] = m[a][b] * (-1 if exactly one of a, b is the third axis else +1)
```

**Notes** — The name is a fossil: it refers to the collision library the engine originally
handed these matrices to, which used the opposite handedness on the third axis. **Nothing in
the current tree calls it.** A rebuild does not need it, but should recognize the shape,
because the same two-sided sign flip is the correct way to convert a rotation between
left- and right-handed frames and the problem recurs at every foreign boundary.

## The product family

**Contract** — Four products, all writing into this matrix from two others, none of which may
alias the destination. The names encode which operand is transposed:

| Operation | Computes |
|---|---|
| product | `A × B` |
| transposed-first product | `Aᵀ × B` |
| transposed-second product | `A × Bᵀ` |
| product plus a vector | `A × B` with one vector component added to every entry of each row |

**Notes** — The transposing forms exist because a rotation's inverse is its transpose and
materializing the transpose first would cost a copy. They are the reason the type has no
inverse for rotations: you transpose instead.

The last form is strange and worth naming as such: it adds the vector's first component to
every entry of the first row, the second to every entry of the second row, and so on. That is
not a meaningful operation on transforms — it is a bulk arithmetic helper, and it is **dead**
in the current tree. A rebuild should not reproduce it without a caller to justify it.

## `Mqinverse` — the adjugate

**Contract** — Writes the adjugate of another matrix: the transposed matrix of cofactors,
which is the inverse multiplied by the determinant. Does **not** divide by the determinant,
which is what the name's "quasi" is admitting.

```text
FUNCTION adjugate(m) -> Matrix3
  FOR EACH (i, j) IN 0..2 × 0..2
    # The cofactor of the transposed position, by the cyclic 2x2 minor.
    result[i][j] = m[(j+1) mod 3][(i+1) mod 3] * m[(j+2) mod 3][(i+2) mod 3]
                 - m[(j+1) mod 3][(i+2) mod 3] * m[(j+2) mod 3][(i+1) mod 3]
```

**Notes** — Leaving the determinant out is useful when the caller is going to divide by
something anyway, or when only the direction of the result matters. It is a trap for anyone
who reads "inverse" in the name. Dead in the current tree.

## `MskewV` — the cross-product matrix

**Contract** — Writes the skew-symmetric matrix that, multiplied by a vector, produces the
cross product with the given vector. Zero diagonal.

```text
FUNCTION cross_matrix(v) -> Matrix3
  RETURN rows
    (  0,   -v.z,   v.y )
    (  v.z,  0,    -v.x )
    ( -v.y,  v.x,   0   )
```

**Invariants** — Under this type's column-vector convention, `cross_matrix(a) × b` equals
`a × b`. The sign convention is therefore tied to the convention warning above; under the
4×4 type's row convention the same nine numbers give the cross product with the arguments
reversed.

**Notes** — This is the standard way to write an angular-velocity or a torque-arm term as a
linear map, which is what a rigid-body solver needs it for. Dead in the current tree — the
physics module went to the library's own routines instead.

## The matrix-vector family

**Contract** — Nine shapes over the same product, all writing into a caller's vector and
none allocating. They vary along three axes: whether the matrix is transposed, whether a
scalar multiplies the result, and whether a second vector is added to or subtracted from it.

| Shape | Computes |
|---|---|
| matrix times vector | `M × v` |
| matrix times vector plus vector | `M × v + u` |
| scaled matrix times vector plus vector | `s × (M × v) + u` |
| transposed times vector | `Mᵀ × v` |
| transposed times vector plus vector | `Mᵀ × v + u` |
| transposed times vector minus vector | `Mᵀ × v − u` |
| scaled transposed times vector | `s × (Mᵀ × v)` |

**Notes** — The naming scheme and the fused shapes are inherited from the rigid-body
library's own vocabulary, which is what the physics bridge was written against. The fusion
exists so a solver step never materializes an intermediate vector. A rebuild in a tier with
expression templates or a decent optimizer writes the expressions out and gets the same code;
a rebuild that writes each step as a separate statement in a tier that allocates temporaries
does not, and that is the decision these names encode.

## `transform_dir` — the odd one out

**Contract** — Transforms a direction by the **first two rows only**, using the *row-vector*
convention: the third component of the input is ignored entirely and the output takes all
three components from the first two rows.

```text
FUNCTION transform_planar_direction(m, v) -> (real, real, real)
  # v.z is not read. This lifts a direction in the plane spanned by the
  # first two basis rows into space.
  RETURN ( v.x*m[1][1] + v.y*m[2][1],
           v.x*m[1][2] + v.y*m[2][2],
           v.x*m[1][3] + v.y*m[2][3] )
```

**Notes** — This contradicts every other vector operation on the type in two ways at once —
the convention and the dropped component — and it shares its name with the 4×4 type's
operation, which drops neither. It is the planar-lift used where a two-dimensional direction
must be expressed in a three-dimensional frame. A rebuild should give it a name that says so
and should not let it sit in the same family as the products above.

## `Meigen` — Jacobi eigen-decomposition

**Contract** — Diagonalizes a **symmetric** matrix by cyclic Jacobi rotations. Writes the
three eigenvalues into a caller's vector and the eigenvectors into this matrix as its
columns. Returns the number of sweeps taken, up to a hard ceiling of 50. **Destroys its
input** — the source matrix is annihilated in place as the rotations are applied, so callers
must pass a copy they do not need. Allocation-free.

```text
FUNCTION eigen_decompose(a) -> (eigenvalues, eigenvectors, sweeps)
  eigenvectors = identity
  diagonal = the diagonal of a
  accumulated = zero                  # pending diagonal corrections
  FOR sweep FROM 0 TO 49
    off_diagonal_sum = |a[1][2]| + |a[1][3]| + |a[2][3]|
    IF off_diagonal_sum = 0 THEN RETURN (diagonal, eigenvectors, sweep)

    # For the first three sweeps, only attack entries above a threshold --
    # annihilating the small ones early wastes rotations that later sweeps
    # would undo. After that, attack everything.
    threshold = IF sweep < 3 THEN 0.2 * off_diagonal_sum / 9 ELSE 0

    FOR EACH off-diagonal entry (p, q) IN ((1,2), (1,3), (2,3))
      IF the entry is negligible against both its diagonal neighbours
         AND at least four sweeps have passed THEN
        set it to zero and skip          # it has converged
      ELSE IF |entry| > threshold THEN
        compute the rotation that zeroes this entry
        apply it to the remaining off-diagonal entry of the matrix
        apply it to all three rows of the eigenvector matrix
        move the annihilated magnitude from the entry onto the two diagonals
        set the entry to zero

    diagonal = diagonal + accumulated   # fold in the pending corrections
    accumulated = zero
  RETURN (diagonal, eigenvectors, 50)   # did not converge; reported by the count
```

**Invariants**

- **The input must be symmetric.** Only the three entries above the diagonal are ever
  examined; the three below are neither read nor updated. Feeding a non-symmetric matrix
  produces an answer that is the decomposition of its symmetric part, silently.
- Convergence is reported through the return value, not through a failure. A return of 50
  means the algorithm ran out of sweeps and the result is whatever it had reached. The
  original logs nothing — the diagnostic is commented out — so a rebuild that wants to know
  must check the count at every call site.
- The pending-corrections accumulator exists so that the diagonal is updated by a sum of
  comparable magnitudes once per sweep, rather than by many small increments. Dropping it
  and updating the diagonal directly is numerically worse for nearly-degenerate tensors,
  which is exactly the case this is used on.

**Notes** — The convergence tests are written as `|d| + g == |d|` — that is, "is `g` so small
relative to `d` that adding it changes nothing at this precision". That is a *precision-
relative* test with no constant in it, and it is the right idea to carry into a rebuild:
the threshold scales with the working precision automatically. The two magic numbers that
remain — three sweeps of thresholding, and a threshold of one fifth of the mean off-diagonal
magnitude — are from the standard published form of the algorithm and are not derived from
anything in this engine.

This is what turns a rigid body's inertia tensor into principal axes and principal moments.
**Nothing in the current tree calls it**, which means the physics module went to the
rigid-body library's own inertia handling instead; a rebuild that keeps its own rigid bodies
will want it back.

## Validity

**Contract** — All nine entries finite and normal.
