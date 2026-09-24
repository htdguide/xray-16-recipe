# src/xrCore/_plane.h

> The plane: a normal and a signed offset, with the classify / project / intersect vocabulary that visibility frustums, portal clipping and triangle-level collision are all written in.

**Needs** — [`_vector3d.h`](_vector3d.h.md) · [`_matrix.h`](_matrix.h.md) · [`math_constants.h`](math_constants.h.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md)

**Used by** — [`level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`level_graph_vertex_inline.h`](../xrAICore/Navigation/level_graph_vertex_inline.h.md) · [`Frustum.cpp`](../xrCDB/Frustum.cpp.md) · [`Frustum.h`](../xrCDB/Frustum.h.md) · [`_plane2.h`](_plane2.h.md) · [`_stl_extensions.h`](_stl_extensions.h.md) · [`vector.h`](vector.h.md) · [`xr_collide_form.h`](../xrEngine/xr_collide_form.h.md)

**Tier floor** — T1: four reals, passed by value and stored in arrays of six inside every frustum, evaluated millions of times per frame in the visibility walk.

## Purpose

Almost every "which side" question in the engine is a plane test: is this bounding sphere
inside the frustum, is this vertex in front of the portal, which side of this polygon's
supporting plane does this point lie on, where does this line of sight cross this wall. A
frustum is six of these; a portal's clip is a handful more.

The whole type is four numbers and one operation — the signed distance from a point to the
plane — with everything else built on it.

## State

```text
RECORD Plane
  normal : (real, real, real)
  offset : real
```

Four reals in that order.

**Invariants**

- The plane is the set of points where `dot(normal, p) + offset = 0`. With a **unit** normal
  the expression is the signed distance from the point to the plane, which is what makes
  every test on this page a comparison against a real distance rather than against an
  arbitrary scale. Almost everything here assumes a unit normal; only the normalize operation
  establishes it.
- The **offset is negative** for a plane whose normal points away from the origin: it is
  minus the projection of any point on the plane onto the normal. Getting this sign backwards
  inverts every inside/outside test in the engine at once.
- The positive side — where the signed distance is positive — is the side the normal points
  to. This type fixes nothing beyond that; which side a user calls "inside" is the user's
  convention, not the plane's. The engine's frustum, the heaviest user, points its normals
  **outward** and therefore treats "inside" as *non-positive against every plane* — see
  [`xrCDB/Frustum.cpp`](../xrCDB/Frustum.cpp.md), which is authoritative on it. Take the
  convention from the user, never assume one here.

## `classify` — the signed distance

**Contract** — The dot product of the point with the normal, plus the offset. Positive in
front, negative behind, zero on the plane. Pure, four multiplies and three adds, and the
single most-executed operation in the visibility layer.

**Notes** — Everything else on this page is one or two of these. A rebuild that optimizes
one thing about the plane type should optimize this, and should consider evaluating four
planes at once against one point, which is what the frustum's inner loop actually wants.

## `distance`

**Contract** — The absolute value of the classification: unsigned distance to the plane.

## Construction

**Contract** — Four ways to build a plane, all allocation-free:

| Form | Behaviour |
|---|---|
| from three points | normal is the normalized cross product of two edges; offset from the first point |
| from three points, precise | the same with the exact normalization, which reports whether the triangle was degenerate |
| from a point and a normal | normalizes the given normal |
| from a point and a **unit** normal | asserts the normal is unit and skips the normalization |

**Invariants** — The three-point forms take edges as `first − second` and `first − third`,
so the resulting normal's direction is fixed by the winding of the three points. Reversing
any two arguments flips the plane. Every caller that builds a plane from a triangle is
relying on the level data's winding convention, so a rebuild that changes winding anywhere
must change it here too.

**Notes** — The precise form exists for the collision and portal code, where a degenerate
triangle is not a rendering artefact but a solver input that produces a normal of length
zero and then infinities. The ordinary form's normalization does not report degeneracy; the
precise one's does. Which to use is a real decision and the choice is made per call site.

## `normalize`

**Contract** — Scales the normal to unit length and scales the offset by the same factor, so
the plane is unchanged as a set of points but the classification becomes a true distance.
Divides by the normal's length without checking it — a zero normal yields infinities.

## `project` — the nearest point on the plane

**Contract** — Writes the point on the plane nearest a given point: the point moved along the
negated normal by its signed distance. Requires a unit normal. Pure.

## `transform` — move a plane by a transform

**Contract** — Rotates the normal by the transform's linear part and slides the offset by the
transform's translation. Allocation-free, in place.

```text
FUNCTION move_plane(plane, transform) -> Plane
  plane.normal = transform_direction(plane.normal, transform)   # no translation
  plane.offset = plane.offset - dot(transform.position, plane.normal)
  RETURN plane
```

**Invariants** — Correct only for a **rigid** transform. The normal is transformed by the
forward linear part, which is the correct transformation for a normal only when the linear
part is orthonormal; a transform carrying non-uniform scale requires the inverse transpose,
and this does not compute it. The offset update uses the *new* normal, after rotation — doing
it in the other order is a silently different plane.

## `intersectRayDist` and `intersectRayPoint` — ray against the plane

**Contract** — Both take a ray origin and direction and report whether the ray reaches the
plane going forward; one writes the distance, the other writes the hit point. Return false
when the direction is within a tight epsilon of parallel to the plane. Pure,
allocation-free.

```text
FUNCTION ray_meets_plane(origin, direction) -> optional<real>
  numerator   = classify(origin)          # signed distance to the plane
  denominator = dot(normal, direction)    # rate of approach
  IF |denominator| < tight_epsilon THEN RETURN none    # parallel
  t = -numerator / denominator
  IF t < 0 AND t is not within epsilon of zero THEN RETURN none   # behind us
  RETURN t
```

**Invariants** — A hit *exactly at* the origin counts as a hit: the acceptance test is
"positive, or near enough to zero". That matters for a ray starting on a surface, which is
the common case when tracing a reflection or a second bounce, and is the reason the test is
not a plain comparison against zero.

The returned distance is along the given direction, so a non-unit direction yields a
parameter, not a distance. Nothing normalizes.

## `intersect` — segment against the plane

**Contract** — Takes two endpoints of a segment, writes the crossing point, returns whether
the segment crosses. Uses the loose epsilon, not the tight one, and admits crossings
slightly outside both ends of the segment.

```text
FUNCTION segment_meets_plane(u, v) -> optional<point>
  edge = v - u
  denominator = dot(normal, edge)
  IF |denominator| < loose_epsilon THEN RETURN none      # parallel to the plane
  t = -(dot(normal, u) + offset) / denominator
  IF t < -epsilon OR t > 1 + epsilon THEN RETURN none    # outside the segment
  RETURN u + edge * t
```

**Notes** — The parameter is normalized to the segment, running zero to one, unlike the ray
forms whose parameter is in units of the direction. Two different parameter conventions in
one type; the segment's is the useful one for clipping a polygon against a plane, which is
what it is for.

## `intersect_2` — the second segment form

**Contract** — A second segment-versus-plane routine with different semantics, kept
alongside the first.

```text
FUNCTION segment_meets_plane_2(u, v) -> optional<point>
  d1 = classify(u)
  d2 = classify(v)
  IF d1 * d2 < 0 THEN RETURN none         # <-- see the note
  RETURN u + (v - u) * (d1 / |d1 - d2|)
```

**Notes** — **This routine's guard is inverted relative to what its shape implies, and a
rebuild should not copy it without deciding what it wants.** Two endpoints on opposite sides
of a plane give a negative product, which is precisely the case where a segment *does* cross
— and that is the case this rejects. It then interpolates by `d1 / |d1 − d2|`, using an
absolute value that makes the parameter's sign independent of which side the first point is
on. Read literally it returns a point for segments that do not cross and refuses the ones
that do.

It survives because the handful of callers use it for a different question than the name
suggests — extrapolating to the plane from two same-side samples — and because the other
form is what the clipping code actually uses. A rebuild should establish what each caller
wanted and then write one segment routine.

## `similar` and `set`

**Contract** — Copy another plane; compare two planes with independent epsilons for the
normal and the offset. The two epsilons are separate parameters because the normal is
dimensionless and the offset is a distance, so one tolerance cannot serve both — a genuinely
useful distinction that most geometry libraries get wrong.

## Validity

**Contract** — Normal and offset all finite and normal.
