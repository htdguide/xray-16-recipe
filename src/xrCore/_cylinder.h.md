# src/xrCore/_cylinder.h

> The finite capped cylinder: a centre, an axis, a height and a radius — the shape the engine uses for anything that is roughly a pole, a limb or a column.

**Needs** — [`_vector3d.h`](_vector3d.h.md) · [`_cylinder.cpp`](_cylinder.cpp.md)

**Used by** — [`Bone.hpp`](Animation/Bone.hpp.md) · [`_cylinder.cpp`](_cylinder.cpp.md) · [`vector.h`](vector.h.md) · [`xr_collide_form.h`](../xrEngine/xr_collide_form.h.md)

**Tier floor** — T1: eight consecutive reals passed by value through picking and collision loops; it is a value type, not an object.

## Purpose

Declares the record and the two intersection surfaces implemented in
[`_cylinder.cpp`](_cylinder.cpp.md). The cylinder is the coarsest useful bound for an
elongated shape — a ladder rung, a tree trunk, a bone's collision proxy — where a sphere is
far too loose and a box has the wrong symmetry.

## State

```text
RECORD Cylinder
  centre    : (real, real, real)   # the midpoint of the axis, not an end cap
  direction : (real, real, real)   # the axis; expected unit length
  height    : real                 # the FULL height, cap to cap
  radius    : real
```

**Invariants**

- `centre` is the middle of the cylinder, so the caps sit at `centre ± direction * height/2`.
  Every routine works with half the height; a rebuild that stores the half-height instead
  removes a division from the inner loop and must then convert wherever a cylinder is read
  from data.
- `direction` must be unit length. Nothing normalizes it and nothing checks it; a non-unit
  axis silently scales the returned ray parameters.
- The "invalidated" cylinder is all zeros, which is degenerate in every field at once —
  zero axis, zero height, zero radius. It is a marker meaning "unset", not a legal shape,
  and feeding it to an intersection is undefined; the orthonormal basis construction inside
  the intersection divides by the axis length.

## Exported units

- **`invalidate`** — zero every field, marking the cylinder unset.
- **Ray classification (`ecode`)** — which surface a returned hit lies on: a flat end
  **cap**, the curved **wall**, or **none**. The caller needs this because a cap hit and a
  wall hit have different normals, and the normal is what the picking caller actually wanted.
- **Ray result (`ERP_Result`)** — the same three-way classification the sphere and the box
  use: no hit, origin inside, origin outside. Sharing the vocabulary across the shapes is
  what lets a picking loop treat them uniformly.
- **`intersect(start, dir, out_parameters, out_codes)`** — the full form: up to two ray
  parameters with a surface code for each. Described in
  [`_cylinder.cpp`](_cylinder.cpp.md).
- **`intersect(start, dir, distance)`** — the narrowing form used when walking a list of
  candidates.
- **Validity** — every field finite and normal.
