# src/Layers/xrRender/blenders/blender_light_reflected.h

> Declares the indirect-light accumulation templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_light_reflected.cpp`](blender_light_reflected.cpp.md)
**Used by** — [`blender_light_reflected.cpp`](blender_light_reflected.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_light_reflected.cpp`](blender_light_reflected.cpp.md).

Exported units:

- **`CBlender_accum_reflected`** — internal template, no class identifier, neither detailable nor lightmappable; emits the single indirect-light accumulation pass.
- **`CBlender_accum_reflected_msaa`** — the same, compiled for one multisample sample index; exists only on backends that do per-sample shading.
