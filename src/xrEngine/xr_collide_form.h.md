# src/xrEngine/xr_collide_form.h

> Declares the per-object collision proxies and the query vocabulary; the substance is in [`xr_collide_form.cpp`](xr_collide_form.cpp.md).

**Needs** — [`xr_collide_form.cpp`](xr_collide_form.cpp.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`xrCore/_obb.h`](../xrCore/_obb.h.md) · [`xrCore/_cylinder.h`](../xrCore/_cylinder.h.md) · [`xrCore/_sphere.h`](../xrCore/_sphere.h.md) · [`xrCore/_plane.h`](../xrCore/_plane.h.md)
**Used by** — [`dxObjectSpaceRender.cpp`](../Layers/xrRender/dxObjectSpaceRender.cpp.md) · [`dxObjectSpaceRender.h`](../Layers/xrRender/dxObjectSpaceRender.h.md) · [`Feel_Vision.cpp`](Feel_Vision.cpp.md) · [`ICollidable.cpp`](ICollidable.cpp.md) · [`ICollidable.h`](ICollidable.h.md) · [`cf_dynamic_mesh.cpp`](cf_dynamic_mesh.cpp.md) · [`cf_dynamic_mesh.h`](cf_dynamic_mesh.h.md) · [`xr_collide_form.cpp`](xr_collide_form.cpp.md) · [`BreakableObject.cpp`](../xrGame/BreakableObject.cpp.md) · [`CustomZone.cpp`](../xrGame/CustomZone.cpp.md) · [`HairsZone.cpp`](../xrGame/HairsZone.cpp.md) · [`HangingLamp.cpp`](../xrGame/HangingLamp.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](../xrGame/Level_bullet_manager_firetrace.cpp.md) · [`MosquitoBald.cpp`](../xrGame/MosquitoBald.cpp.md) · _and 13 more_
**Tier floor** — T1.

## Purpose

Declares the surface described in [`xr_collide_form.cpp`](xr_collide_form.cpp.md), plus the
query vocabulary that is *only* declared here.

Exported units:

- **`ICollisionForm`** — the interface every proxy satisfies: a ray query, the owner, and
  the local bounding box and sphere the object space rejects against. Carries a kind tag
  distinguishing an object's own form from a standalone shape volume, which the object
  space branches on.
- **`CCF_Skeleton`** — the per-bone proxy, with its element list, the per-element centre
  query, and the lookup of one element by bone index (a binary search over the list, which
  is why the list is kept ordered by bone).
- **`CCF_EventBox`** — the six-plane trigger volume and its containment test.
- **`CCF_Shape`** — the authored sphere-and-box list, its builders, its bounds computation
  and its containment test.
- **`clQueryCollision`** and **`clQueryTri`** — the result record for a shape query:
  affected objects, triangles, boxes and spheres, with helpers that transform each into
  world space as it is appended.
- **The query flag set** — which shape kinds to collect, first-hit-only, top-level-only,
  static, dynamic, and a coarse mode that tests triangles against an oriented box. Only the
  ray-query flags defined in the collision database are actually honoured; this set belongs
  to the box query that was never implemented.

## Notes

The bone element packs its three shape variants into overlapping storage, so the record is
the size of its largest variant (a transform plus a vector). That matters because a
skeleton has tens of elements and every animated object on screen has a skeleton. A rebuild
should use a tagged union and will get the same size; what it must not do is give each
element room for all three.
