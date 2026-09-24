# src/Layers/xrRender/blenders/dx11MinMaxSMBlender.cpp

> The pass that builds the min/max acceleration texture over the sun's shadow map.

**Needs** — [`dx11MinMaxSMBlender.h`](dx11MinMaxSMBlender.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — [`dx11MinMaxSMBlender.h`](dx11MinMaxSMBlender.h.md) · [`glMinMaxSMBlender.cpp`](glMinMaxSMBlender.cpp.md)
**Tier floor** — T2: one pass description.

## Purpose

Filtering a shadow map with a wide kernel costs many samples per pixel. Most of those samples are wasted: over most of the screen a whole neighbourhood of the shadow map is entirely nearer than the receiver or entirely farther, and the answer is "fully lit" or "fully shadowed" without sampling at all.

This pass precomputes, for each tile of the shadow map, the minimum and maximum depth in that tile. The sun accumulation passes that name a `_minmax` pixel program read it first and skip the expensive filter when the tile's range settles the question. See the `SUN_NEAR_MINMAX` element in [`blender_light_direct.cpp`](blender_light_direct.cpp.md).

## `CBlender_createminmax.compile(context)`

**Contract** — emits one full-screen pass, element zero only, reading the shadow depth target and writing the min/max target. No depth, no blending.

```text
FUNCTION compile(C)
  base_compile(C)
  IF C.element != 0 THEN RETURN
  C.pass(vertex="stub_notransform_2uv", pixel="create_minmax_sm",
         fog=false, depth_test=false, depth_write=false, blend=false)
  C.depth(test=false, write=false, not inverted)
  C.texture("s_smap", the sun shadow depth target)   # point-sampled
  C.end()
```

**Invariants** — the shadow map must be read as a plain depth texture here, *not* through a comparison sampler. The pass wants the stored depth values themselves; a comparison sampler would return the result of a test against a reference and the reduction would be meaningless. On the OpenGL backend the same binding is expressed as a comparison-mode sampler with the comparison disabled — see [`glMinMaxSMBlender.cpp`](glMinMaxSMBlender.cpp.md) — which is the same thing said differently.

The tile size is fixed by the pixel program and by the min/max target's dimensions relative to the shadow map's; neither is stated here, and both must agree with the accumulation programs that consume the result.
