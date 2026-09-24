# src/Layers/xrRender/blenders/blender_light_point.h

> Declares the point-light accumulation templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_light_point.cpp`](blender_light_point.cpp.md)
**Used by** — [`blender_light_point.cpp`](blender_light_point.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_light_point.cpp`](blender_light_point.cpp.md).

Exported units:

- **`CBlender_accum_point`** — internal template, no class identifier, neither detailable nor lightmappable; emits the five quality levels of an omnidirectional light's accumulation pass.
- **`CBlender_accum_point_msaa`** — the same, compiled for one multisample sample index; exists only on backends that do per-sample shading.
