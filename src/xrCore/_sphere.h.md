# src/xrCore/_sphere.h

> The sphere: centre and radius, with the ray, sphere and point tests that the visibility, audio and collision layers all use as their cheap first check.

**Needs** — [`_vector3d.h`](_vector3d.h.md) · [`math_constants.h`](math_constants.h.md) · [`_sphere.cpp`](_sphere.cpp.md)
**Used by** — [`ISpatial.h`](../xrCDB/ISpatial.h.md) · [`Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [`Bone.hpp`](Animation/Bone.hpp.md) · [`_sphere.cpp`](_sphere.cpp.md) · [`vector.h`](vector.h.md) · [`vis_common.h`](../xrEngine/vis_common.h.md) · [`xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`ShapeData.h`](../xrServerEntities/ShapeData.h.md)
**Tier floor** — T1: it is a four-float value whose layout is relied on, passed by value through the inner loops of visibility and collision; a tier that boxes it turns every test into a pointer chase.

## Purpose

Almost every spatial question in the engine starts with a sphere: is this object's bounding sphere in the frustum, does this shot's ray reach this creature, can this sound be heard here. The tests live in the header because they are inlined into those loops.

## State

```text
RECORD Sphere
  centre : (real, real, real)
  radius : real
```

No invariant is enforced. A negative radius is meaningful to no test here and is not checked; an "empty" sphere is expressed by the caller, not by the type. The identity value is the unit sphere at the origin, not an empty one — which is a trap if a caller expects `identity` to mean "nothing".

## Ray intersection — the full form

**Contract** — Takes a ray origin, a **unit** direction, and a range that scales that direction, and reports where along the ray the sphere is entered and left. Returns three things: a classification (no hit, origin inside, origin outside), a count of returned parameters, and up to two parameters along the ray. Allocation-free, pure.

The classification is the useful part and is what callers branch on:

- **no hit** — the ray misses, or both intersections are behind the origin;
- **origin outside** — both roots exist and the nearer one is ahead; two parameters are returned, entry then exit;
- **origin inside** — the nearer root is behind the origin and the farther is ahead; **one** parameter is returned, the exit point, placed in the first slot.

```text
FUNCTION intersect(origin, direction, range) -> (classification, count, t[2])
  # Solve the quadratic for |origin + t*direction*range - centre| = radius,
  # written so the returned t values are already scaled by `range` --
  # callers compare them against a distance, not against a unit parameter.
  diff = origin - centre
  a = range * range
  b = dot(diff, direction) * range
  c = square_magnitude(diff) - radius * radius
  discriminant = b*b - a*c

  IF discriminant < 0 THEN RETURN (no hit, 0, -)
  IF discriminant > 0 THEN
    root = sqrt(discriminant)
    t[0] = range * (-b - root) / a           # near
    t[1] = range * (-b + root) / a           # far
    IF t[0] >= 0 THEN RETURN (origin outside, 2, t)
    IF t[1] >= 0 THEN t[0] = t[1]; RETURN (origin inside, 1, t)
    RETURN (no hit, 0, -)
  # Tangent: one root.
  t[0] = range * (-b / a)
  IF t[0] >= 0 THEN RETURN (origin outside, 1, t)
  RETURN (no hit, 0, -)
```

**Invariants** — The direction must be unit length; `range` carries the magnitude. Feeding a non-unit direction scales the returned parameters silently.

**Notes** — The tangent case reports "origin outside" with a single root. A caller that assumes "outside implies two roots" is wrong on the measure-zero case; the count is there to be read.

## Ray intersection — the narrowing forms

**Contract** — Two convenience shapes over the above, both taking the current best distance by reference and narrowing it:

- **nearest-hit form** — returns the classification only when the near hit is *closer than* the distance passed in, and writes that distance back. This is the one used when walking a list of candidates, and it is why callers do not need to compare afterwards.
- **clamping form** — always reports the classification, and narrows the distance only for the inside and outside cases, leaving it alone when the hit is farther than the caller's current best.

**Notes** — The two differ in whether a hit *behind* the current best is reported at all. Both exist because both behaviours are wanted in different callers, and choosing the wrong one produces a picking bug that only shows when two objects overlap.

## Ray intersection — the algebraic shortcut

**Contract** — A second formulation that skips the quadratic: project the centre onto the ray, compare the perpendicular distance to the radius, and take the near root directly. Reports inside or outside by whether the origin is within the radius, and narrows a caller's distance. Cheaper, and used where only the near hit matters.

**Notes** — The two formulations do not agree bit for bit and the engine uses both. That is a fact about it, not a decision: a rebuild may use one.

## Boolean tests

**Contract** — Four cheap predicates, all pure and allocation-free:

- **ray hits at all** — perpendicular-distance test, no parameters produced, no behind-the-origin check. It answers "does the infinite line come within the radius", which is not the same question as the parameterized form answers.
- **spheres overlap** — squared centre distance against the squared sum of radii. Touching exactly is *not* an overlap.
- **contains a point** — squared distance against the squared radius, with a small epsilon added so a point exactly on the surface counts as inside.
- **contains a sphere** — false immediately when the argument is larger; otherwise the squared centre distance against the squared radius difference.

## `volume`

**Contract** — Four-thirds pi times the radius cubed.

## Validity

**Contract** — A sphere is valid when its centre and radius are all finite, normal numbers — see [`_std_extensions.h`](_std_extensions.h.md) for what that excludes. Used in assertions throughout the physics and visibility layers.

## `Fsphere_compute`

**Contract** — Declared here, implemented in [`_sphere.cpp`](_sphere.cpp.md): the smallest sphere enclosing a set of points.
