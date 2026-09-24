# src/Layers/xrRender/blenders/Blender_Lm(EbB).h

> Declares the lightmapped reflective world-surface template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a declaration only.

## Purpose

Declares the template implemented in [`Blender_Lm(EbB).cpp`](Blender_Lm%28EbB%29.cpp.md). Unlike most templates in this directory, one implementation serves every renderer generation, with the generation-specific elements compiled out.

## Exported units

- **`CBlender_LmEbB`** — the lightmapped-reflective class tag at parameter version 1. Lightmappable, not detailable. Three parameters: the environment texture, its matrix, and a blend flag.
