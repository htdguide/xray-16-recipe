# src/Layers/xrRender/blenders/glMSAABlender.cpp

> The edge-marking pass, as the OpenGL backend binds it.

**Needs** — [`dx11MSAABlender.h`](dx11MSAABlender.h.md) · [`dx11MSAABlender.cpp`](dx11MSAABlender.cpp.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: one pass description.

## Purpose

The same template as [`dx11MSAABlender.cpp`](dx11MSAABlender.cpp.md) — same class, same program names, same state — differing only in that a texture and its sampling mode are bound as one combined unit rather than as a texture plus a separate sampler object. Exactly one of the two files is built.

This file exists because the original's two backends model texture binding differently. **In a rebuild with one binding model it does not exist**: implement [`dx11MSAABlender.cpp`](dx11MSAABlender.cpp.md) and delete this one.

## `CBlender_msaa.compile(context)`

**Contract** — as in the file above. The two g-buffer targets are bound with the combined "render-target, point-filtered" sampler form.
