# src/xrEngine/cf_dynamic_mesh.h

> Declares the collision form that refines a skeleton hit down to the exact triangle.

**Needs** — [`cf_dynamic_mesh.cpp`](cf_dynamic_mesh.cpp.md) · [`xr_collide_form.h`](xr_collide_form.h.md)
**Used by** — [`cf_dynamic_mesh.cpp`](cf_dynamic_mesh.cpp.md)
**Tier floor** — T1: sits directly on the collision query path, which is measured per shot

## Purpose

Declares the surface implemented in [`cf_dynamic_mesh.cpp`](cf_dynamic_mesh.cpp.md).

Exported units:

- `CCF_DynamicMesh` — a collision form over an animated skeleton. Inherits the
  bone-volume form's registration, bounds and box queries unchanged, and replaces only the
  ray query with one that follows up each bone-volume hit by testing the bone's actual
  triangles.
