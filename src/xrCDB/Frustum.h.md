# src/xrCDB/Frustum.h

> Declares the frustum type — the convex volume of half-spaces the visibility pass
> and the collision queries share.

**Needs** — [`xrCDB.h`](xrCDB.h.md) · [`xrCore/FixedVector.h`](../xrCore/FixedVector.h.md) · [`xrCore/_plane.h`](../xrCore/_plane.h.md)
**Used by** — [`Frustum.cpp`](Frustum.cpp.md) · [`ISpatial_q_frustum.cpp`](ISpatial_q_frustum.cpp.md) · [`xrCDB_frustum.cpp`](xrCDB_frustum.cpp.md) · [`xr_area_query.cpp`](xr_area_query.cpp.md) · [`Render.h`](../xrEngine/Render.h.md) · [`xr_collide_form.cpp`](../xrEngine/xr_collide_form.cpp.md) · [`dbg_draw_frustum.cpp`](../xrGame/dbg_draw_frustum.cpp.md)
**Tier floor** — T2: it fixes a capacity so the type is a stack value, which most tiers can
express.

## Purpose

Declares the surface implemented in [`Frustum.cpp`](Frustum.cpp.md). Three things are
decided here rather than there:

- **the three-way visibility answer** — *outside*, *partially inside*, *fully inside*. Every
  hierarchical traversal in the engine branches on all three, so a two-valued test would not
  do: *fully inside* is what lets a traversal stop testing a plane for a whole subtree.
- **the plane budget** — twelve, and the polygon buffer at four times that. Both are fixed
  so a frustum and a clip buffer are values, not allocations; the reasoning is in
  [`Frustum.cpp`](Frustum.cpp.md).
- **the plane-selection flags** for extracting a frustum from a matrix, named per side plus
  the two useful groupings (the four sides, and all six).

## State

Declares the shape; the record and its invariants are in
[`Frustum.cpp`](Frustum.cpp.md).

## Exported units

- **`Frustum`** — the type; operations listed in [`Frustum.cpp`](Frustum.cpp.md).
- **`Visibility`** — the three-way answer.
- **`Polygon`** — a fixed-capacity vertex list, the currency of every clipping operation here.
- **the box-corner table** — eight rows of six, mapping a plane normal's sign pattern to the
  two box corners its test needs. Defined in the implementation; declared here because the
  box-against-plane test is inline.
- **the box-against-plane test** and **polygon containment**, both inline in the header
  because they sit inside the visibility pass's inner loop.
