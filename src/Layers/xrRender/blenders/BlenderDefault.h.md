# src/Layers/xrRender/blenders/BlenderDefault.h

> Declares the lightmapped-diffuse world-surface template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`BlenderDefault.cpp`](BlenderDefault.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the template implemented in [`BlenderDefault.cpp`](BlenderDefault.cpp.md).

## Exported units

- **`CBlender_default`** — identity is the lightmapped-diffuse class tag at parameter version 1. Answers **yes** to "can be detailed" and "can be lightmapped", which is what lets the material compiler resolve a detail texture for it and what tells the level compiler to bake a lightmap for surfaces using it. Carries one parameter, a four-way tessellation selector, which the shipping renderers read and ignore.
