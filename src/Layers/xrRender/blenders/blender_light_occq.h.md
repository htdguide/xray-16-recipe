# src/Layers/xrRender/blenders/blender_light_occq.h

> Declares the occlusion-query template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_light_occq.cpp`](blender_light_occq.cpp.md)
**Used by** — [`blender_light_occq.cpp`](blender_light_occq.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_light_occq.cpp`](blender_light_occq.cpp.md).

Exported units:

- **`CBlender_light_occq`** — internal template, no class identifier, neither detailable nor lightmappable; emits the query draw, the stencil marker write and the marker-block reset.
