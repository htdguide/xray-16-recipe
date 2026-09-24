# src/Layers/xrRender_R2/r3_rendertarget_create_minmaxSM.cpp

> Reduces a cascade's shadow map to a quarter-size min/max image so the light-shaft march
> can skip depth ranges that hold no occluder.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`xrRender/blenders/dx11MinMaxSMBlender.h`](../xrRender/blenders/dx11MinMaxSMBlender.h.md)
**Used by** — [`r4_rendertarget_accum_direct.cpp`](../xrRenderPC_R4/r4_rendertarget_accum_direct.cpp.md)
**Tier floor** — T1: a full-screen draw with explicit depth and stencil state.

## Purpose

Light shafts are computed by stepping along the view ray and asking the shadow map, at
each step, whether that point is lit. Most steps land in regions where the answer cannot
change — the whole depth interval is either in front of every occluder or behind all of
them. This pass precomputes, for each 4×4 block of the shadow map, the minimum and maximum
depth it contains, so the march can test a whole block at once and skip it.

## `create_minmax_shadow_map`

**Contract** — draws one full-screen triangle pair into the quarter-size target with the
reduction material, reading the cascade currently in the atlas. Depth test always passes,
depth writes off, no culling, stencil off. Leaves colour writes enabled.

```text
FUNCTION create_minmax_shadow_map()
  fill a quad spanning clip space, with the source's texture coordinates
  bind the quarter-size min/max target
  material = the reduction description
  depth: always pass, no write; no culling; stencil off
  draw
```

**Notes** — the target is a quarter of the atlas's side, so each output texel summarises a
4×4 block; the reduction is one pass, not a pyramid, because a single level is all the
march uses. The two values are packed into one two-channel float target, which is why it
is created as a single-channel format only in the degenerate configuration — see
[`r2_rendertarget.cpp`](r2_rendertarget.cpp.md).

This runs only for the near cascade, and only when
[the min/max decision](r2_rendertarget.cpp.md#need_light_shafts--use_minmax_this_frame)
says the resolution and the weather make it worthwhile. The near cascade is the one the
march spends its steps in.
