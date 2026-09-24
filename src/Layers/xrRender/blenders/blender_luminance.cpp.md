# src/Layers/xrRender/blenders/blender_luminance.cpp

> The three-step reduction that turns the frame down to a single average-luminance texel, with temporal smoothing against the previous frame's answer.

**Needs** — [`blender_luminance.h`](blender_luminance.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — [`blender_luminance.h`](blender_luminance.h.md)
**Tier floor** — T2: a decision tree that emits pass descriptions, with the target sizes fixed by the render-target set.

## Purpose

Auto-exposure needs one number per frame: how bright is what the camera is looking at. Reading that off the GPU would stall, so it is computed *on* the GPU by successive downsampling and consumed as a texture by the tone-mapping pass a frame later. This template describes the three reduction steps.

## `CBlender_luminance.compile(context)`

**Contract** — emits one pass per element, each a full-screen draw with no depth and no blending, reading one scratch target and writing the next.

```text
FUNCTION compile(C)
  base_compile(C)
  SWITCH C.element
    CASE 0:  # 256x256 -> 64x64
      C.pass(vertex="stub_notransform_build", pixel="bloom_luminance_1",
             fog=false, depth_test=false, depth_write=false, blend=false)
      C.texture("s_image", bloom target 1)   # linearly filtered
      C.end()
    CASE 1:  # 64x64 -> 8x8
      C.pass(vertex="stub_notransform_filter", pixel="bloom_luminance_2", ...)
      C.texture("s_image", luminance scratch 64)
      C.end()
    CASE 2:  # 8x8 -> 1x1, blended with the previous frame's result
      C.pass(vertex="stub_notransform_filter", pixel="bloom_luminance_3", ...)
      C.texture("s_image",   luminance scratch 8)
      C.texture("s_tonemap", the PREVIOUS frame's luminance result)
      C.end()
```

**Invariants**

- The chain's source is the **bloom** target, not the scene target. Bloom has already downsampled the frame to 256×256 and the luminance chain reuses that work; a rebuild that changes the bloom resolution changes the reduction's first step with it.
- The sizes 256 → 64 → 8 → 1 are fixed by the render-target set and by the reduction factor each pixel program performs (4× then 8× then 8×). The programs and the target sizes must agree; neither is free.
- Every sample in steps 0 and 1 is *linearly* filtered, which is how a 4× or 8× reduction reads sixteen or sixty-four texels with four or sixteen fetches. Point sampling here would compute the average of a sparse subset.
- The third step reads the previous frame's result and the new one, and writes the blend. That temporal smoothing is why the exposure eases rather than snapping when the player walks out of a building; the smoothing rate lives in the pixel program, not here. The previous and current luminance targets must therefore be two distinct surfaces that the renderer swaps between frames.

**Notes** — Two different vertex programs appear, `_build` and `_filter`. The first emits the tap offsets for a 4× reduction and the second for an 8× one; they are not interchangeable, and which one pairs with which pixel program is part of the frozen shader set.
