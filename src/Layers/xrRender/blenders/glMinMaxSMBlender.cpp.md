# src/Layers/xrRender/blenders/glMinMaxSMBlender.cpp

> The shadow-map min/max reduction, as the OpenGL backend binds it.

**Needs** — [`dx11MinMaxSMBlender.h`](dx11MinMaxSMBlender.h.md) · [`dx11MinMaxSMBlender.cpp`](dx11MinMaxSMBlender.cpp.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: one pass description.

## Purpose

The same template as [`dx11MinMaxSMBlender.cpp`](dx11MinMaxSMBlender.cpp.md) — same class, same program names, same state — differing only in how the shadow map is bound: here through the depth-comparison sampler form, there as a plain texture with a point sampler. Exactly one of the two files is built.

**In a rebuild with one texture-binding model this file does not exist**: implement the other and delete this one.
