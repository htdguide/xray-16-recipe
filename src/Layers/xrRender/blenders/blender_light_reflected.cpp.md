# src/Layers/xrRender/blenders/blender_light_reflected.cpp

> The template that adds the one-bounce indirect term to the accumulator.

**Needs** — [`blender_light_reflected.h`](blender_light_reflected.h.md) · [`blender_light_direct.cpp`](blender_light_direct.cpp.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — [`blender_light_reflected.h`](blender_light_reflected.h.md)
**Tier floor** — T2: a decision tree that emits pass descriptions.

## Purpose

The simplest template in the lighting set: one pass, one element, no branching on quality. It stands in for light that has bounced once off a surface — the renderer treats it as a virtual light with a volume, so the pass is drawn as a volume and reads the same four g-buffer inputs every accumulation pass reads.

## `CBlender_accum_reflected.compile(context)`

**Contract** — emits exactly one pass, whatever the element index is. Allocates nothing.

```text
FUNCTION compile(C)
  base_compile(C)
  blend = device_can_blend_accumulator_format
  dest  = blend ? ONE : ZERO

  C.pass(vertex="accum_volume", pixel="accum_indirect",
         fog=false, depth_test=false, depth_write=false,
         blend=blend, src=ONE, dst=dest)
  C.texture("s_position",    g-buffer position)
  C.texture("s_normal",      g-buffer normal)
  C.texture("s_material",    material lookup)
  C.texture("s_accumulator", the accumulator)
  C.end()
```

**Invariants** — unlike the sun template, this one *does* honour the device's ability to blend the accumulator's format rather than forcing it off. The sun forces it off because a filter pass runs afterwards; the indirect term has no such pass.

**Notes** — The element index is read nowhere, so any element compiles the same pass. That is not an oversight: the caller only ever asks for element zero, and the template is small enough that adding a switch would be noise.

`CBlender_accum_reflected_msaa` is the same body with the `_msaa`-suffixed program and a baked sample index.
