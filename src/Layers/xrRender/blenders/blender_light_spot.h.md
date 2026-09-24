# src/Layers/xrRender/blenders/blender_light_spot.h

> Declares the cone-light accumulation templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_light_spot.cpp`](blender_light_spot.cpp.md)
**Used by** — [`blender_light_spot.cpp`](blender_light_spot.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_light_spot.cpp`](blender_light_spot.cpp.md).

Exported units:

- **`CBlender_accum_spot`** — internal template, no class identifier, neither detailable nor lightmappable; emits the five quality levels of a cone light's accumulation pass.
- **`CBlender_accum_spot_msaa`** — the same, compiled for one multisample sample index.
- **`CBlender_accum_volumetric_msaa`** — the in-air beam pass for one cone light.

The latter two exist only on backends that do per-sample shading.
