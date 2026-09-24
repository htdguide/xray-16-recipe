# src/Layers/xrRender/blenders/blender_deffer_aref.h

> Declares the deferred filling of the alpha-tested world-surface class tag.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`blender_deffer_aref.cpp`](blender_deffer_aref.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the template implemented in [`blender_deffer_aref.cpp`](blender_deffer_aref.cpp.md). It claims the **same class tag** as [`Blender_default_aref.h`](Blender_default_aref.h.md) — alternative fillings of one identifier, selected by the backend's class table.

## Exported units

- **`CBlender_deffer_aref`** — the alpha-tested class tag at parameter version 1. **Constructed with a lightmapped flag**, so the renderer registers this one class twice, once for the lightmapped tag and once for the vertex-lit one; the flag is the answer to the lightmap capability query and selects which program the blended path uses. Detailable and parallax-capable in both forms. Two parameters: an alpha reference defaulting to **200** (where the forward filling of the same tag defaults to 32) and a blend flag.
