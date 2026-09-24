# src/Layers/xrRender/blenders/blender_light_mask.h

> Declares the stencil-mask and accumulator-copy templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_light_mask.cpp`](blender_light_mask.cpp.md)
**Used by** — [`blender_light_mask.cpp`](blender_light_mask.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_light_mask.cpp`](blender_light_mask.cpp.md).

Exported units:

- **`CBlender_accum_direct_mask`** — internal template, no class identifier, neither detailable nor lightmappable; emits the six no-colour passes of the masking and accumulator-copy element namespace.
- **`CBlender_accum_direct_mask_msaa`** — the same, compiled for one multisample sample index; exists only on backends that do per-sample shading.
