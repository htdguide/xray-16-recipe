# src/Layers/xrRender/blenders/Blender_Screen_SET.h

> Declares the general-purpose "basic" material template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`Blender_Screen_SET.cpp`](Blender_Screen_SET.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the template implemented in [`Blender_Screen_SET.cpp`](Blender_Screen_SET.cpp.md). One implementation serves every renderer generation.

## Exported units

- **`CBlender_Screen_SET`** — the basic class tag at parameter version 4. Seven parameters, which together are the pass state: a ten-way blend-mode selector (index order frozen), a texture-clamp flag, an alpha reference, and four booleans for depth test, depth write, lighting and fog. No capabilities declared: no detail, no lightmap.
