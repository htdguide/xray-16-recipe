# src/Layers/xrRender/blenders/blender_light_direct_cascade.cpp

> The sun-accumulation template for the cascade shadow scheme that keeps one shadow map per cascade instead of one shared atlas.

**Needs** — [`blender_light_direct_cascade.h`](blender_light_direct_cascade.h.md) · [`blender_light_direct.cpp`](blender_light_direct.cpp.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — [`blender_light_direct_cascade.h`](blender_light_direct_cascade.h.md)
**Tier floor** — T2: a decision tree that emits pass descriptions.

## Purpose

A second sun template, differing from [`blender_light_direct.cpp`](blender_light_direct.cpp.md) only in which pixel programs it names. It exists because the engine supports two sun-shadow schemes and the choice is made per frame by the renderer, not per material: a rebuild that implements one scheme only will have one of these two files and not the other.

## `CBlender_accum_direct_cascade.compile(context)`

**Contract** — emits the near/middle and far sun accumulation passes, using the pixel programs `accum_sun_cascade` and `accum_sun_cascade_far`. Shares the element namespace, the binding set, the inverted-depth rule for the near cascades and the white-shadow-border rule for the far one with the file above; those decisions are documented there and not repeated.

```text
FUNCTION compile(C)
  base_compile(C)
  blend = false ; dest = ZERO           # forced, as in the non-cascade template

  SWITCH C.element
    CASE SUN_NEAR, SUN_MIDDLE:
      C.pass(vertex="accum_volume", pixel="accum_sun_cascade",
             depth_test=true, depth_write=false, blend=blend, src=ONE, dst=dest)
      C.depth(test=true, write=false, INVERTED)
      bind s_position, s_normal, s_material, s_accumulator, s_lmap, s_smap, jitter
      C.end()
    CASE SUN_FAR:
      C.pass(vertex="accum_volume", pixel="accum_sun_cascade_far", ...)
      ... same bindings ...
      C.sampler_address("s_smap", BORDER) ; C.sampler_border("s_smap", opaque white)
      C.end()
```

**Notes** — The commented-out border override on the near cascades is a real difference, not dead code: a near cascade's footprint is fully inside the scene, so a pixel outside it is handled by the *next* cascade, not by the border. Only the last cascade needs the "everything beyond is lit" answer.

The luminance and min/max elements are absent here. This scheme feeds the tone-mapper from the other template's luminance pass, and has no min/max acceleration texture.
