# src/xrPhysics/tri-colliderknoopc/dTriColliderMath.h

> Point-in-triangle, point-across-plane, and the one routine that prepares a triangle for
> testing.

**Needs** — [`__aabb_tri.h`](__aabb_tri.h.md) · [`dcTriangle.h`](dcTriangle.h.md) · [`../MathUtilsOde.h`](../MathUtilsOde.h.md) · [`../../xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md)
**Used by** — [`CalculateTriangle.h`](../CalculateTriangle.h.md) · [`dSortTriPrimitive.h`](dSortTriPrimitive.h.md) · [`dTriColliderCommon.h`](dTriColliderCommon.h.md) · [`dTriSphere.cpp`](dTriSphere.cpp.md)
**Tier floor** — T2: triangle predicates. T1 in the original only because they are inlined
into the collider's innermost loop.

## Purpose

Header-only. Three predicates and one constructor, used by every one of the three
primitive-versus-triangle routines. They are collected here because all four depend on the
same convention — the triangle's normal points out of its front face and its edges are stored
as two vectors, not three — and separating them from that convention would make them
meaningless.

## Stateless.

## `TriContainPoint`

**Contract** — does a point, projected onto the triangle's plane, lie inside the triangle?

```text
FUNCTION contains(v0, v1, v2, normal, edge_0, edge_1, edge_2, point) -> bool
  FOR EACH (vertex, edge) IN ((v0, edge_0), (v1, edge_1), (v2, edge_2))
    inward := cross(normal, edge)              # points into the triangle
    IF dot(inward, point) < dot(inward, vertex)
      RETURN false                             # outside this edge
  RETURN true
```

Three forms exist, differing only in how much has already been computed: all three edges
given, two given (the third derived), or nothing given (all derived from the vertices).

**Invariants** — the test is on the *infinite prism* through the triangle, not the triangle
itself: a point far above or below the plane still counts as contained. That is deliberate and
every caller relies on it — the question being asked is always "is the shape over the face, or
over an edge or a corner", and the answer must not depend on distance.

**Notes** — the three overloads exist purely so that a caller who has already paid for the
edges does not pay again. In a rebuild the prepared triangle record carries them and there is
one form.

## `TriPlaneContainPoint`

**Contract** — is a point in front of the triangle's plane? Three forms again: from the three
vertices, from a precomputed normal and one vertex, or straight from the prepared triangle's
stored signed distance.

**Notes** — the last form — "is the stored distance positive" — is the one the hot loop
actually uses. The other two exist for callers who have not prepared the triangle yet.

## `PlanePoint`

**Contract** — given a prepared triangle, a segment from one point to another that crosses its
plane, and the signed distance of the *from* end, produce the crossing point.

```text
FUNCTION crossing(tri, from, to, from_distance) -> point
  span := tri.distance_of(to) - from_distance     # must be negative: the ends differ in sign
  RETURN from - (to - from) * (from_distance / span)
```

**Invariants** — the caller must already know the segment crosses the plane; the routine
asserts it rather than checking. This is the swept test's core operation: a shape's centre
moved from `from` to `to` during the step, and this is where it went through the triangle's
plane. Calling it on a segment that does not cross produces a point off the segment.

## `InitTriangle` and `CalculateTri`

**Contract** — fill a prepared triangle from a collision-database triangle and the vertex
array it indexes: compute the two edges, the normalised normal and the plane offset
(`InitTriangle`), and additionally the signed distance of a given position from that plane
(`CalculateTri`).

Two forms of each exist: one taking the vertex array and the triangle's indices, one taking
three already-fetched vertices. The second exists because the collider's inner loop fetches the
vertices once and then uses them for the extent test, the preparation and the contact
generation.

**Invariants** — the normal is normalised here and only here. Everything downstream assumes
unit length: depths are computed as differences of projections along it, and an unnormalised
normal scales every penetration depth by the triangle's area, which the solver reads as a much
deeper collision than occurred.

**Notes** — `CalculateTri` takes the *shape's centre* as its position, so the stored distance
is the centre's distance, not the surface's. Each primitive converts that into a penetration
by subtracting its own half-extent along the normal — which is exactly what the three
`Proj` routines in [`dcTriListCollider.h`](dcTriListCollider.h.md) provide. That split — the
mesh side computes the distance, the primitive side computes its own reach — is what lets one
traversal serve three different primitives.
