# src/xrPhysics/CalculateTriangle.h

> Turns a triangle from the level's collision soup into the plane, sides and
> signed distance the mesh collider works from, and answers the point-to-triangle question
> that decides face, edge or vertex contact.

**Needs** — [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`Geometry.h`](Geometry.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`tri-colliderknoopc/dTriColliderMath.h`](tri-colliderknoopc/dTriColliderMath.h.md) · [`xrCDB/xr_area.h`](../xrCDB/xr_area.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`CustomRocket.cpp`](../xrGame/CustomRocket.cpp.md) · [`wallmark_manager.cpp`](../xrGame/wallmark_manager.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md)
**Tier floor** — T1: it reads triangles out of the level's memory-image collision data by
index and computes on raw float runs, in the hottest loop of the collision bridge.

## Purpose

The static collision database stores a triangle as three indices into a shared vertex array
and nothing else — no normal, no plane, no edges
([chapter 7](../xrCDB/README.md)). The mesh collider needs all of those, for every candidate
triangle, on every step. This file is the conversion, and the working record it produces.

It also answers the question the whole contact-generation design turns on: **which feature
of this triangle is nearest to this point — its face, one of its edges, or one of its
vertices?** Getting that answer wrong is what makes an object catch on a flat floor, so it
is worth reading this page before any of the `tri-colliderknoopc` ones.

## State

```text
RECORD Triangle              # the working form; one per candidate triangle per step
  side0 : vector             # vertex1 - vertex0
  side1 : vector             # vertex2 - vertex1
  norm  : vector             # unit normal, = normalize(cross(side0, side1))
  pos   : real               # the plane constant: dot(vertex0, norm)
  dist  : real               # SIGNED distance from the query point to the plane
  depth : real               # penetration depth, filled in later by the collider
  tri   : Triangle reference # back-pointer into the level's soup: material, flags, sector
  # invariant: winding fixes the normal's sense. A point with dist > 0 is on the
  #            triangle's FRONT.  Everything downstream assumes that.
```

## `CalculateInitTriangle`

**Contract** — given a soup triangle and the shared vertex array, fill in the two side
vectors, the unit normal and the plane constant. The third side is not stored: it is
recovered when needed as the closing edge, because storing it would grow the record for a
value used in one of three branches.

**Notes** — the normal is normalized here, once, and every consumer relies on that. This is
the call that meets a degenerate (zero-area) triangle first, which is why the normalize it
uses must be the careful one — see [`MathUtilsOde.h`](MathUtilsOde.h.md).

## `CalculateTriangle`

**Contract** — the above, plus the signed distance from a query point to the triangle's
plane. Two forms: one taking a point directly, one taking a collision shape and using that
shape's true world position, which saves every caller the transform-unwrapping dance.

## `TriContainPoint`

**Contract** — does the point project inside the triangle? Returns yes, or no plus **which
edge it fell outside of**.

```text
FUNCTION tri_contain_point(v0, v1, v2, normal, side0, side1, point) -> (bool, int)
  side2 = v0 - v2                         # the closing edge, recovered here
  FOR EACH (vertex, side, code) IN [(v0,side0,1), (v1,side1,2), (v2,side2,3)]
    outward = cross(normal, side)         # points out of the triangle in its plane
    IF dot(outward, point) < dot(outward, vertex)
      RETURN (false, code)                # outside THIS edge; stop, the code is the answer
  RETURN (true, 0)
```

**Invariants** — the returned code identifies the *first* edge the point failed against, and
the caller depends on that identification, not merely on the yes/no. The test short-circuits,
so a point outside two edges reports only the first — which is correct, because the nearest
feature in that case is the vertex those two edges share, and the follow-up distance
computation discovers exactly that.

**Notes** — there is no tolerance. A point exactly on an edge is reported inside, since the
comparison is strict in the other direction. That asymmetry is deliberate: it guarantees
that two triangles sharing an edge cannot *both* reject a point lying on it, so a contact is
never lost in the crack between them. Losing a contact there is precisely how an object
falls through a floor.

## `DistToFragmenton`

**Contract** — distance from a point to a line segment, reporting the closest point on the
segment, the direction to the query point, and **which part of the segment was closest**:
the first endpoint, the second endpoint, or the interior.

```text
FUNCTION dist_to_segment(point, a, b) -> (distance, closest, direction, region)
  v = b - a
  t = dot(point - a, v) / dot(v, v)        # parameter of the perpendicular foot
  IF t < 0    RETURN (|point - a|, a, a - point, endpoint_a)
  IF t > 1    RETURN (|point - b|, b, b - point, endpoint_b)
  foot = a + v * t
  RETURN (|point - foot|, foot, point - foot, interior)
```

**Notes** — the region code is the second half of the feature classification. Combined with
the edge code from the containment test, it names one of seven features exactly: the face,
one of three edges, or one of three vertices.

## `DistToTri`

**Contract** — the composed answer. Given the working triangle and a point, classify the
nearest feature and return the distance to it plus the outward direction.

```text
FUNCTION dist_to_tri(T, point) -> (distance, direction, closest_point, classification)
  IF point is behind the triangle's plane
    RETURN (-1, _, _, BEHIND)            # reject: contact only from the front

  (inside, edge_code) = tri_contain_point(T, point)
  IF inside
    RETURN (T.dist, -T.norm, point - T.norm * T.dist, ON_PLANE)

  (d, closest, dir, region) = dist_to_segment(point, the two vertices of edge_code)
  IF region is interior
    RETURN (d, normalize(dir), closest, ON_EDGE)

  # fell off an end of that edge: the nearest feature is a VERTEX
  vertex = the endpoint region names
  RETURN (|vertex - point|, normalize(vertex - point), vertex, ON_VERTEX)
```

**Invariants** — a point behind the plane is rejected outright with a negative distance. This
is the one-sidedness of the whole static collider: level geometry is a shell with an inside
and an outside, and an object that has got behind a wall is not pushed back out through it.
That is why the *degeneracy* handling in `tri-colliderknoopc` matters so much — once an
object is on the wrong side of a triangle, nothing here will save it.

**Notes** — distances degenerate to zero rather than to a normalized direction when the
point coincides with the feature; the direction is then left as the zero vector and callers
must not normalize it. This is the file's one silent contract and it is not asserted
anywhere.

The three classifications are not just diagnostic. The collider treats them differently:
a face contact takes the triangle's own normal, an edge contact takes the direction to the
edge, and a vertex contact takes the direction to the vertex — and only the face case may
use the triangle's material and flags without further thought, because an edge or vertex is
shared with a neighbouring triangle whose material may differ.
