# src/Layers/xrRender/blenders/Blender_detail_still_deferred.cpp

> The detail-object template filled for the deferred renderers, where the cut-out edges can be resolved against multisampling instead of against a hard alpha test.

**Needs** — [`Blender_detail_still.h`](Blender_detail_still.h.md) · [`uber_deffer.h`](uber_deffer.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The deferred renderers answer the detail-object class tag with this file instead of [`Blender_detail_still.cpp`](Blender_detail_still.cpp.md); exactly one is built, and everything but `Compile` is duplicated verbatim.

The wind decision carries over unchanged: the high-quality element names the swaying vertex program and the low-quality one names the still program. What is added is the alpha-to-coverage path — a way to antialias the grass blades' cut-out edges when multisampling is on, which costs a second pass.

## `Compile`

```text
FUNCTION compile(context)
  vertex program := element is high quality ? "detail_w" : "detail_s"
  atoc := the device is resolving alpha tests through coverage

  SELECT context.element
    normal_hq, normal_lq ->
      IF atoc THEN
          # PASS 1: depth and coverage only. Same geometry, the coverage
          # pixel program, colour writes off, alpha-to-coverage on. This
          # pass establishes which samples the blade covers.
          shared deferred emission with pixel program "<base>_atoc"
          mark the stencil, disable colour writes, cull nothing
          enable alpha-to-coverage
          end the pass
      # PASS 2 (or the only pass): the g-buffer write.
      shared deferred emission with pixel program "base"
      mark the stencil, cull nothing
      IF atoc THEN require depth EQUAL rather than less-or-equal
      end the pass
```

**Invariants**

- Backface culling is **off** for detail objects in every case. A grass card is a two-sided sheet; culling it would make half the blades vanish depending on the camera's side.
- When the coverage path is taken, the second pass tests depth for *equality* against what the first pass wrote. That is what confines the g-buffer write to exactly the samples the coverage pass accepted — it is stencil testing done with the depth buffer, because the stencil is already carrying the g-buffer coverage mark and cannot carry this too.
- Both passes mark the stencil with the g-buffer value, under the mask that preserves the high bit, exactly as every other g-buffer-writing template does.

**Notes** — The oldest deferred renderer has no coverage path at all: its `Compile` is two lines, one per element, with no stencil marking either. The feature and the stencil mark arrived together with the generation that introduced multisampling, and a rebuild targeting one device model implements only the shape that device needs.

The blend parameter this template carries is never consulted on this path. A detail object in a deferred renderer is always a g-buffer write, and a g-buffer write cannot blend.
