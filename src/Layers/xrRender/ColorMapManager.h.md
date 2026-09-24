# src/Layers/xrRender/ColorMapManager.h

> Declares the holder of the two colour-grading lookup textures.

**Needs** — [`SH_Texture.h`](SH_Texture.h.md)
**Used by** — [`ColorMapManager.cpp`](ColorMapManager.cpp.md) · [`gl_rendertarget.h`](../xrRenderPC_GL/gl_rendertarget.h.md) · [`r4_rendertarget.h`](../xrRenderPC_R4/r4_rendertarget.h.md) · [`r2_rendertarget_phase_PP.cpp`](../xrRender_R2/r2_rendertarget_phase_PP.cpp.md)
**Tier floor** — T2: a two-slot cache over named textures.

## Purpose

Declares the surface implemented in [`ColorMapManager.cpp`](ColorMapManager.cpp.md).

## Exported units

- **`ColorMapManager`** — owns two named texture slots that shaders sample as colour-grading lookups, plus a cache of every grading texture ever bound, keyed by name.
