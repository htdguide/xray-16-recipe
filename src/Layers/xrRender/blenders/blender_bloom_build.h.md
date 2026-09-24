# src/Layers/xrRender/blenders/blender_bloom_build.h

> Declares the bloom chain's and the final composite's internal templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`blender_bloom_build.cpp`](blender_bloom_build.cpp.md) · [`r2_rendertarget_phase_bloom.cpp`](../../xrRender_R2/r2_rendertarget_phase_bloom.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the templates implemented in [`blender_bloom_build.cpp`](blender_bloom_build.cpp.md). All three answer **no** to both capability queries and carry no parameters; their identity is the zero class tag, meaning they are constructed by the engine and never looked up from the material library.

## Exported units

- **`CBlender_bloom_build`** — the five-element bloom chain: transfer into the bloom target, then a separable horizontal and vertical blur, then the two wide-kernel alternatives. Present in every generation.
- **`CBlender_bloom_build_msaa`** — the same five elements again, declared only in the generations that support multisampling and currently identical to the above.
- **`CBlender_postprocess_msaa`** — the final composite, with two elements: plain, and colour-graded through a pair of gradient maps.
