# src/Layers/xrRender/blenders/blender_light_direct_cascade.h

> Declares the alternate sun-accumulation template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_light_direct_cascade.cpp`](blender_light_direct_cascade.cpp.md)
**Used by** — [`blender_light_direct_cascade.cpp`](blender_light_direct_cascade.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_light_direct_cascade.cpp`](blender_light_direct_cascade.cpp.md).

Exported units:

- **`CBlender_accum_direct_cascade`** — internal template, no class identifier, neither detailable nor lightmappable; emits the near/middle and far sun passes for the per-cascade shadow scheme.
