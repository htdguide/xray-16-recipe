# src/Layers/xrRender/blenders/Blender_BmmD.h

> Declares the terrain template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`Blender_BmmD.cpp`](Blender_BmmD.cpp.md) · [`Blender_BmmD_deferred.cpp`](Blender_BmmD_deferred.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares one template with **two alternative implementations**, exactly one of which is built: [`Blender_BmmD.cpp`](Blender_BmmD.cpp.md) for the forward renderer and [`Blender_BmmD_deferred.cpp`](Blender_BmmD_deferred.cpp.md) for the deferred ones.

## Exported units

- **`CBlender_BmmD`** — the terrain class tag at parameter version 3. Detailable, lightmappable, and parallax-capable. Six parameters: the tiled detail texture and its matrix, plus four ground textures, one per channel of the derived mask. The four exist only for the deferred fillings; the forward one loads and ignores them.
