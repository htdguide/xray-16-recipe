# src/Layers/xrRender/blenders/Blender_LaEmB.h

> Declares the lightmap-plus-environment template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`Blender_LaEmB.cpp`](Blender_LaEmB.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the template implemented in [`Blender_LaEmB.cpp`](Blender_LaEmB.cpp.md), whose six private emissions are the same composition fitted to different texture-stage budgets.

## Exported units

- **`CBlender_LaEmB`** — the lightmap-alpha-environment class tag. Lightmappable, not detailable, and takes no dynamic light. Three parameters: the environment texture, its matrix, and an optional constant multiplier whose absence is spelled `"$null"`.
