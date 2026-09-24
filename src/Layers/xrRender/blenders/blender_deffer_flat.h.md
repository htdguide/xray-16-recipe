# src/Layers/xrRender/blenders/blender_deffer_flat.h

> Declares the deferred filling of the lightmapped-diffuse class tag.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`blender_deffer_flat.cpp`](blender_deffer_flat.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the template implemented in [`blender_deffer_flat.cpp`](blender_deffer_flat.cpp.md). It claims the **same class tag** as [`BlenderDefault.h`](BlenderDefault.h.md) — the two are alternative fillings of one identifier, selected by which backend's class table is in effect.

## Exported units

- **`CBlender_deffer_flat`** — the lightmapped-diffuse class tag at parameter version 1. Detailable and parallax-capable; **not** lightmappable, unlike the forward filling of the same tag. One parameter: a four-way tessellation selector, which on one backend is the only place in the engine that selector reaches a shader.
