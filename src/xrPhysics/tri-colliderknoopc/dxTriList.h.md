# src/xrPhysics/tri-colliderknoopc/dxTriList.h

> What the dynamics library stores for a mesh shape, and the per-triangle and per-batch
> callbacks a caller may hang off it.

**Needs** — [`dcTriListCollider.h`](dcTriListCollider.h.md) · [`dTriList.cpp`](dTriList.cpp.md) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`dTriList.cpp`](dTriList.cpp.md) · [`dTriList.h`](dTriList.h.md) · [`dcTriListCollider.cpp`](dcTriListCollider.cpp.md)
**Tier floor** — T1: it declares the payload layout the dynamics library allocates inside a
user shape.

## Purpose

The level's static geometry is one collision shape as far as the solver is concerned. This
file says what that shape carries: a plane (unused vestigially), two optional callbacks, and
the collider object that does the actual work.

It also carries a vector type and a plane type of its own, which are incidental — they came
with the triangle-list code this was adapted from — and a pair of comparison helpers.

## State

```text
RECORD MeshShape                  # the dynamics library's payload for the mesh shape
  plane           : four reals    # vestigial; the mesh has no plane
  per_triangle_cb : optional<callback>
  per_batch_cb    : optional<callback>
  collider        : MeshCollider  # owned; destroyed with the shape
```

**Invariants** — exactly one mesh shape exists per level, and it owns its collider. The
collider holds the per-step candidate cache, so destroying the shape at level unload is what
frees it.

## the two callbacks

**Contract** — a caller may register either or both:

```text
CALLBACK per_triangle(mesh, other_shape, triangle_index) -> accept : bool
    # consulted for one triangle at a time; a false answer drops it

CALLBACK per_batch(mesh, other_shape, triangle_indices, count)
    # told about the whole candidate set at once, for bookkeeping
```

**Notes** — neither is used by the engine. They are part of the interface this code arrived
with, and they survive because the shape's payload layout is declared to the dynamics library
by size and changing it means touching the registration. A rebuild deletes them: per-triangle
filtering is done by material flags inside the traversal
([`dSortTriPrimitive.h`](dSortTriPrimitive.h.md)), and there is nothing a batch callback could
usefully do.

The *contact* callback that the engine does use is a different thing entirely and lives on the
colliding shape's user data, not here — see [`../ExtendedGeom.h`](../ExtendedGeom.h.md).

## the vector and plane types

**Contract** — a three-float vector with the usual arithmetic, dot and cross products,
magnitude and normalisation; a plane built from three points, carrying a normal and a
distance, able to say whether a point is in front of it within a tolerance.

**Invariants** — the plane's containment test uses a *non-strict* comparison, so a point
exactly on the plane counts as contained. The original marks the change from a strict
comparison explicitly. It matters because a shape resting perfectly on a surface is the common
case, not the rare one, and the strict form makes it flicker in and out of contact.

**Notes** — the vector type exists only so that this code could be adopted without rewriting
it, and it duplicates the engine's own. A rebuild has one vector type and none of this.

The two min/max helpers are likewise the standard library the file was written without.
