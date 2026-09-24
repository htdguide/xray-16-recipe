# src/xrCore/_plane2.h

> The line in the plane: a two-dimensional normal and a signed offset — the 2D twin of the plane type, used where a "which side" question is genuinely planar.

**Needs** — [`_vector2.h`](_vector2.h.md) · [`math_constants.h`](math_constants.h.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md) · [`_plane.h`](_plane.h.md)

**Used by** — [`vector.h`](vector.h.md)

**Tier floor** — T1: three reals passed by value, shaped like the 3D plane so the same idioms read in both.

## Purpose

A half-plane test in two dimensions: which side of this line is this point on, where does
this segment cross it. The engine wants it for the navigation mesh's planar work and for
screen-space clipping, where carrying a third coordinate through every comparison is
meaningless.

Despite the name this is a **line**, not a plane. The name is chosen so the 2D and 3D types
read as a pair, and the operation names and semantics are deliberately identical to
[`_plane.h`](_plane.h.md) — which is the useful part: a reader who knows one knows the other.

## State

```text
RECORD Line2
  normal : (real, real)
  offset : real
```

Three reals in that order. The invariants are the 3D plane's: the line is where
`dot(normal, p) + offset = 0`, the offset is negative for a normal pointing away from the
origin, the positive side is the one the normal points to, and every operation except
`normalize` assumes the normal is unit.

**Notes** — This header does not include the 2-vector type it is built on; it relies on a
translation unit having included it first. That is an ordering hazard in the original and is
purely incidental — a rebuild's module system removes the problem.

## The shared vocabulary

Every operation behaves exactly as the matching one on
[`_plane.h`](_plane.h.md) does, one dimension lower and with the same epsilons:

| Operation | Behaviour |
|---|---|
| `classify` | signed distance from a point; positive on the normal's side |
| `distance` | its absolute value |
| `build` from a point and a normal | normalizes the normal, derives the offset |
| `normalize` | scale the normal to unit length, scaling the offset with it |
| `project` | the nearest point on the line |
| `intersectRayDist` / `intersectRayPoint` | ray against the line; rejects a near-parallel direction by the tight epsilon; a hit at the origin counts |
| `intersect` | segment against the line, parameter normalized to the segment, admitted slightly beyond both ends |
| `intersect_2` | the second segment form, carrying the same inverted guard — see the [3D version's note](_plane.h.md#intersect_2--the-second-segment-form) |
| `similar` / `set` | compare with independent normal and offset epsilons; copy |

**Notes** — Only the construction differs in kind: there is no three-point form, because two
points determine a line and the third has nowhere to be. The 3D type's precise construction
and its transform-by-a-matrix have no counterpart here either, so a 2D line cannot be moved
by a transform without the caller doing it by hand.

The duplication between this file and [`_plane.h`](_plane.h.md) is close to total. That the
two exist as separate hand-written files rather than as one definition over a dimension is an
artifact of the tier; a rebuild writes it once.

## Validity

**Contract** — Normal and offset finite and normal.
