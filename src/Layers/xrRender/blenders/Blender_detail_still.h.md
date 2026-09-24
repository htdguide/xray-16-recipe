# src/Layers/xrRender/blenders/Blender_detail_still.h

> Declares the detail-object (grass and debris) template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`Blender_detail_still.cpp`](Blender_detail_still.cpp.md) · [`Blender_detail_still_deferred.cpp`](Blender_detail_still_deferred.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares one template with **two alternative implementations**, exactly one of which is built: [`Blender_detail_still.cpp`](Blender_detail_still.cpp.md) for the forward renderer and [`Blender_detail_still_deferred.cpp`](Blender_detail_still_deferred.cpp.md) for the deferred ones.

## Exported units

- **`CBlender_Detail_Still`** — the detail-object class tag at parameter version 0. One parameter, a blend flag, which the deferred fillings ignore. No capabilities declared. The wind animation is not a parameter: it is selected by the shader element, so detail quality and wind are one switch.
