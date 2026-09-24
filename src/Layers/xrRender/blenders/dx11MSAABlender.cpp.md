# src/Layers/xrRender/blenders/dx11MSAABlender.cpp

> The pass that finds the frame's geometric edges and marks them in the stencil buffer, so that later passes can shade those pixels per sample and everything else once.

**Needs** — [`dx11MSAABlender.h`](dx11MSAABlender.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — [`dx11MSAABlender.h`](dx11MSAABlender.h.md) · [`glMSAABlender.cpp`](glMSAABlender.cpp.md)
**Tier floor** — T2: one pass description.

## Purpose

Per-sample shading is expensive and only matters where samples within one pixel disagree — that is, at a silhouette or a sharp normal discontinuity. This pass runs once after the g-buffer is filled, compares the samples of the position and normal targets, and writes the **edge marker** stencil bit where they differ.

That bit (`0x80`, bit 7) is the other half of the stencil layout documented in [`blender_light_occq.cpp`](blender_light_occq.cpp.md) and [`blender_deffer_model.cpp`](blender_deffer_model.cpp.md): every pass that writes stencil must leave it alone, and the marker-block reset drops it from its write mask when multisampling is on.

## `CBlender_msaa.compile(context)`

**Contract** — emits one full-screen pass, element zero only. No colour output that matters; the stencil operation and reference are set by the draw code.

```text
FUNCTION compile(C)
  base_compile(C)
  IF C.element != 0 THEN RETURN
  C.pass(vertex="stub_notransform_2uv", pixel="mark_msaa_edges",
         fog=false, depth_test=false, depth_write=false, blend=false)
  C.depth(test=false, write=false, not inverted)
  C.texture("s_position", g-buffer position)   # bound as the MULTISAMPLED surface
  C.texture("s_normal",   g-buffer normal)     # likewise
  C.sampler(point, no filtering)
  C.end()
```

**Invariants** — the two g-buffer targets must be bound as multisampled surfaces, not as resolved ones, or the pass has nothing to compare and marks nothing. Point sampling is not a preference: filtering a multisampled surface is meaningless.

**Notes** — The pixel program discards where the samples agree, which is how the stencil write is made conditional without a second pass. A rebuild on an API where discard and stencil interact differently must arrange the same effect.
