# src/Layers/xrRender/blenders/blender_luminance.h

> Declares the luminance-reduction template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_luminance.cpp`](blender_luminance.cpp.md)
**Used by** — [`blender_luminance.cpp`](blender_luminance.cpp.md) · [`r2_rendertarget_phase_luminance.cpp`](../../xrRender_R2/r2_rendertarget_phase_luminance.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_luminance.cpp`](blender_luminance.cpp.md).

Exported units:

- **`CBlender_luminance`** — internal template, no class identifier, neither detailable nor lightmappable; emits the three steps of the average-luminance reduction.
