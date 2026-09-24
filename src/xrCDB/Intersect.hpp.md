# src/xrCDB/Intersect.hpp

> The analytic intersection predicates the collision layer uses where a tree is not
> involved — ray, box, sphere and oriented box against a single triangle.

**Needs** — [`xrCore/_matrix33.h`](../xrCore/_matrix33.h.md) · [`xrCore/_obb.h`](../xrCore/_obb.h.md) · [`xrCore/_sphere.h`](../xrCore/_sphere.h.md)
**Used by** — [`SkeletonCustom.cpp`](../Layers/xrRender/SkeletonCustom.cpp.md) · [`SkeletonX.cpp`](../Layers/xrRender/SkeletonX.cpp.md) · [`SkeletonX.h`](../Layers/xrRender/SkeletonX.h.md) · [`r2_rendertarget_enable_scissor.cpp`](../Layers/xrRender_R2/r2_rendertarget_enable_scissor.cpp.md) · [`xr_area_raypick.cpp`](xr_area_raypick.cpp.md) · [`xr_efflensflare.cpp`](../xrEngine/xr_efflensflare.cpp.md) · [`ActorCameras.cpp`](../xrGame/ActorCameras.cpp.md) · [`DynamicHeightMap.cpp`](../xrGame/DynamicHeightMap.cpp.md) · [`PHMovementControl.cpp`](../xrGame/PHMovementControl.cpp.md) · [`ik_foot_collider.cpp`](../xrGame/ik_foot_collider.cpp.md) · [`PHSimpleCharacter.cpp`](../xrPhysics/PHSimpleCharacter.cpp.md) · [`dcTriListCollider.cpp`](../xrPhysics/tri-colliderknoopc/dcTriListCollider.cpp.md) · [`SoundRender_Scene.cpp`](../xrSound/SoundRender_Scene.cpp.md)
**Tier floor** — T2: small-vector arithmetic with early exits; nothing here touches layout or
a device.

## Purpose

A tree query answers *which triangles*. These answer *this shape against that triangle*, one
pair at a time, and they are what callers use once the tree has narrowed the candidates —
the ray cache's single remembered triangle, the physics layer's per-body tests, the
engine's own collision-form tests against boxes and spheres.

They are in a header with no implementation file because they are all small, all leaf calls
inside somebody else's inner loop, and none has state. The grouping is the only reason this
is one file rather than five; a rebuild may split or merge freely.

## State

Stateless.

## `ray_hits_sphere`

**Contract** — true when an infinite ray from an origin along a unit direction passes within
a sphere. No distance, no entry point, no range limit. Cheap trivial reject, nothing more.

## `ray_hits_triangle`

**Contract** — intersects a ray with a single triangle, reporting the distance along the ray
and the two barycentric coordinates of the hit. Takes a culling flag: when set, a triangle
whose front face points away from the ray is rejected outright. Reports a hit at any
distance, positive or negative — the range check belongs to the caller.

This is the same Möller–Trumbore test the tree's ray traversal runs at its leaves (see
[`xrCDB_ray.cpp`](xrCDB_ray.cpp.md)), with the same two arms: under culling the barycentric
bounds are compared against the un-normalized determinant so the reciprocal is taken only
for a surviving hit; without culling a determinant near zero either way means the ray lies
in the triangle's plane and is rejected.

**Notes** — the file offers it twice, once over three vertices and once over three pointers
to vertices, which is a calling-convention artifact: some callers hold a triangle, others
hold indices into a shared vertex array. One entry point taking a view of three points
covers both. A second pair reports only the distance, for callers that do not want the
barycentrics.

## `box_hits_triangle`

**Contract** — true when an *oriented* box overlaps a triangle. The box is given as a
rotation, a translation and three half-extents; the triangle as three points in world space.
Takes a culling flag which, when set, rejects a triangle whose normal faces the same way as
the box's third axis — the caller's "this surface faces away from me" rule, not a geometric
one.

The predicate is the separating-axis test, the same thirteen candidate axes as the
axis-aligned version in [`xrCDB_box.cpp`](xrCDB_box.cpp.md): the triangle's plane, the box's
three axes, and the nine cross products of a box axis with a triangle edge. The difference
is that here every projection goes through the box's rotation, so there is no coordinate to
read off directly and the test is several times the cost. The ordering is chosen so the
cheapest rejections come first and each axis is computed only if the previous ones failed
to separate.

The interval comparisons are written as a small family of "is this interval outside
`[-r, r]`" shapes parameterized on how the interval is expressed — as a point plus one
offset, or a point plus two — which is where all the early exits live. That is a shape, not
a decision; the decision is *reject on the first separating axis and compute nothing
further*.

## `sphere_hits_triangle`

**Contract** — classifies a sphere against a triangle as *disjoint*, *intersecting*, or
*triangle wholly inside the sphere*. The three-way answer matters to the caller: a triangle
wholly inside a sphere is a containment, not a contact, and the physics layer handles the
two differently.

```text
FUNCTION sphere_hits_triangle(centre, radius, v0, e0, e1) -> {none, intersect, inside}
  n <- how many of the three vertices lie within the sphere
  IF n == 3: RETURN inside
  IF n  > 0: RETURN intersect
  # all three vertices outside, but the sphere may still touch an edge or the face
  IF squared_distance_point_to_triangle(centre, v0, e0, e1) < radius^2: RETURN intersect
  RETURN none
```

The vertex count is checked first because it decides most cases for three squared lengths.

## `squared_distance_point_to_triangle`

**Contract** — the squared distance from a point to the closest point of a triangle, the
triangle given as an origin and two edge vectors. Exact, and the largest single routine in
the file.

**Invariants** — non-negative; zero exactly when the point lies in the triangle.

Minimizing the quadratic in the triangle's own barycentric parameters partitions the plane
into seven regions — the interior, three edge regions and three vertex regions — and the
routine dispatches on which region the unconstrained minimum falls into, then clamps to that
region's boundary. Each of the seven has its own closed form. It is long because it is a
case analysis, not because it is doing anything deep; the sub-cases are pure algebra and a
rebuild should take them from a geometry reference rather than transcribe them. The load-
bearing part is the *structure*: find the unconstrained minimum, identify its region, clamp.

## `sphere_hits_oriented_box`

**Contract** — true when a sphere overlaps an oriented box. Transforms the sphere's centre
into the box's frame — three dot products — and then reduces to the same face/edge/vertex
case analysis as the point-to-box distance, comparing against the squared radius only in the
corner cases where it is actually needed.

## `ray_hits_oriented_box`

**Contract** — true when a ray meets an oriented box. No distance, no entry point: a
yes/no. Separating-axis again, in the box's frame: three tests on the box's own axes (with
the "and the ray is heading away from the box" refinement that lets a miss be decided from
one dot product pair), then three on the cross products of the ray direction with each box
axis.

## Notes

Nothing here has a range parameter. Every predicate answers about an *infinite* ray or a
complete shape, and every caller applies its own range afterwards. That is consistent and
worth preserving: pushing the range in would double the number of entry points and the
callers already need the distance back for other reasons.
