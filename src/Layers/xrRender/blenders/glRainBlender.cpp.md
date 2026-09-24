# src/Layers/xrRender/blenders/glRainBlender.cpp

> The wet-surface passes, as the OpenGL backend binds them.

**Needs** — [`dx11RainBlender.h`](dx11RainBlender.h.md) · [`dx11RainBlender.cpp`](dx11RainBlender.cpp.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a decision tree that emits pass descriptions.

## Purpose

The same two templates as [`dx11RainBlender.cpp`](dx11RainBlender.cpp.md) — same classes, same elements, same programs, same state, same fixed water-texture paths, same shadow-map-size switch and the same blend-factor override that turns the gloss pass into a multiply. The only difference is that each texture is bound together with its sampling mode as one unit (render-target-filtered, comparison, or ordinary) instead of as a texture plus a separate sampler object.

**In a rebuild with one texture-binding model this file does not exist**: implement [`dx11RainBlender.cpp`](dx11RainBlender.cpp.md) and delete this one. Its decisions are documented there and not repeated.
