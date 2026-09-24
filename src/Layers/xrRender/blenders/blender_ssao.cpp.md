# src/Layers/xrRender/blenders/blender_ssao.cpp

> The templates for the screen-space ambient-occlusion pass and for the half-resolution depth target it reads.

**Needs** — [`blender_ssao.h`](blender_ssao.h.md) · [`blender_light_direct.cpp`](blender_light_direct.cpp.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — [`blender_ssao.h`](blender_ssao.h.md)
**Tier floor** — T2: a decision tree that emits pass descriptions.

## Purpose

Ambient occlusion darkens creases the lighting model cannot see into. It is computed from the g-buffer alone, at reduced resolution, and multiplied into the ambient term later. Two elements: the occlusion pass itself, and a depth downsample that produces the half-resolution depth the occlusion pass samples.

## `CBlender_SSAO_noMSAA.compile(context)`

**Contract** — emits one of two full-screen passes. Neither writes depth or blends.

```text
FUNCTION compile(C)
  base_compile(C)
  SWITCH C.element
    CASE 0:  # compute occlusion
      C.pass(vertex="combine_1", pixel="ssao_calc_nomsaa",
             fog=false, depth_test=false, depth_write=false)
      C.stencil(on, compare=LESS_OR_EQUAL, read_mask=0xFF)
      C.stencil_ref(0x01)                    # only where a surface was written
      C.cull(none)
      C.texture("s_position",   g-buffer position)
      C.texture("s_normal",     g-buffer normal)
      C.texture("s_tonemap",    the current luminance result)
      C.texture("s_half_depth", the half-resolution depth target)
      bind_jitter(C)
      C.end()

    CASE 1:  # downsample depth for the horizon-based variant
      C.pass(vertex="combine_1", pixel="depth_downs",
             fog=false, depth_test=false, depth_write=false)
      C.cull(none)
      C.texture("s_position", g-buffer position)
      C.texture("s_normal",   g-buffer normal)
      C.texture("s_tonemap",  the current luminance result)
      C.end()
```

**Invariants**

- The occlusion pass is stencil-limited to pixels carrying the surface marker (bit 0, reference `0x01`). Sky pixels have no geometry to occlude and computing occlusion for them wastes the most expensive pass in the frame.
- The downsample pass is deliberately *not* stencil-limited: the half-resolution depth must be valid everywhere the occlusion pass's sampling kernel can reach, including one kernel radius outside the stencilled region.
- Binding the luminance result into an occlusion pass looks wrong and is not: the occlusion strength is scaled by the current exposure so that the effect does not swamp a dark scene.

**Notes** — Only the newest generation has the half-resolution depth input and the second element; the oldest deferred generation computes occlusion directly from the full-resolution position target and has no downsample step. A rebuild may implement either.

## `CBlender_SSAO_MSAA.compile(context)`

**Contract** — the occlusion pass only, compiled for one multisample sample index (the mechanism is described in [`blender_light_direct.cpp`](blender_light_direct.cpp.md)). It uses a *different* stencil predicate: equality against `0x81` rather than "at least `0x01`".

**Invariants** — `0x81` is the surface marker (bit 0) **and** the multisample edge marker (bit 7) together, tested for exact equality. Per-sample occlusion is only worth computing on pixels that both contain geometry and straddle an edge; everywhere else the resolved single-sample answer is correct and much cheaper. Reproducing this predicate is what keeps the per-sample path affordable.

## Notes on the file's shape

The header declares the occlusion template under two names — one for the oldest deferred generation, one for the newer ones — and aliases them so callers need not branch. That alias is an artifact of the original's conditional compilation; a rebuild has one template.
