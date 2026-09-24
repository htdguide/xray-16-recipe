# src/Layers/xrRender/blenders/Blender_Model_EbB.h

> Declares the reflective dynamic-model template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`Blender_Model_EbB.cpp`](Blender_Model_EbB.cpp.md) · [`Blender_Model_EbB_deferred.cpp`](Blender_Model_EbB_deferred.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares one template with **two alternative implementations**, exactly one of which is built: [`Blender_Model_EbB.cpp`](Blender_Model_EbB.cpp.md) for the forward renderer and [`Blender_Model_EbB_deferred.cpp`](Blender_Model_EbB_deferred.cpp.md) for the deferred ones. Everything but `Compile` is identical in both, and duplicated rather than shared.

## Exported units

- **`CBlender_Model_EbB`** — the reflective-model class tag at parameter version 1. Three parameters: the environment texture's name, the matrix that generates its coordinates, and a blend flag. Inherits "no" to detail and lightmap capability.
