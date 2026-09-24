# src/xrGame/magic_box3.cpp

> Whether two oriented boxes overlap, and where the eight corners of one are.

**Needs** — [`magic_box3.h`](magic_box3.h.md)
**Used by** — [`magic_box3.h`](magic_box3.h.md)
**Tier floor** — T1: a fixed sequence of dot products in the inner loop of collision queries

## Purpose

An oriented box is the shape the game uses for anything that must be tested cheaply but is
not axis-aligned: a fitted volume around a group of points, a trigger, a hit region. This
file answers the one hard question about two of them — do they overlap — and the one
mechanical question — where are the corners.

## State

`Stateless.`

## `intersects`

**Contract** — reports whether this box and another share any point, treating both as closed
solids. Touching counts as overlapping. Pure; allocates nothing; no early exit beyond the
first separating axis found.

The method is the **separating axis test**: two convex solids are disjoint if and only if
some axis exists on which their projections do not overlap, and for two boxes only fifteen
candidate axes need be tried — each box's three face normals, and the nine cross products of
one box's axis with the other's.

```text
FUNCTION intersects(other) -> bool
  d = other.center - this.center
  c[i][j]    = dot(this.axis[i], other.axis[j])       # 3x3, computed lazily per group
  abs_c[i][j]= |c[i][j]|
  ad[i]      = dot(this.axis[i], d)

  FOR EACH of the 15 candidate axes
    separation = |projection of d onto the axis|
    reach      = (this box's projected half-width) + (other box's projected half-width)
    IF separation > reach THEN RETURN false          # a separating axis exists

  RETURN true
```

**Invariants** — both boxes' axes must be orthonormal, or every projection is scaled by the
axis length and the answer is wrong. The fifteen axes must be tried in full before
concluding overlap; *any one* of them failing to separate proves nothing.

The projected half-width of a box onto an axis is the sum over its three axes of
(half-extent × |cosine between that axis and the test axis|) — which is why the matrix of
absolute dot products is computed once and reused fifteen times. For the nine cross-product
axes the same quantities reappear in a permuted order, so no new dot product is needed
beyond the initial twelve.

**Notes**

- The fifteen tests are written out one after another rather than looped, with the dot
  products for each of this box's axes computed just before its first use. That structure is
  a *short-circuit* optimization, not a style: the first three tests are the ones most likely
  to separate in practice — boxes in this game are usually far apart along an axis of the
  first one — so the remaining dot products are frequently never computed. A rebuild that
  builds the whole matrix up front and then loops is correct and slower on the common case.
- No tolerance is applied anywhere. Two boxes exactly touching along a face are reported as
  overlapping, and near-parallel boxes suffer the classic ill-conditioning of the
  cross-product axes: a cross product of two nearly-parallel axes is nearly zero, and
  comparing its near-zero projections is dominated by rounding. The original accepts this. A
  rebuild that cares should skip a cross-product axis whose length is below a tolerance and
  rely on the face-normal tests, which are exact in that configuration.

## `ComputeVertices`

**Contract** — writes the box's eight corners into a caller-supplied buffer of eight
positions, in a **fixed order**: the four corners of the face at negative third-axis first,
counterclockwise from the all-negative corner, then the four of the opposite face in the same
rotational order.

**Invariant** — the order is part of the contract, not an implementation detail. Callers
index the array to name edges and faces; a rebuild that emits the corners in a different
order silently breaks every one of them.
