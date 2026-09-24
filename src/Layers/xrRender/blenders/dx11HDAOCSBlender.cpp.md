# src/Layers/xrRender/blenders/dx11HDAOCSBlender.cpp

> The two compute-shader ambient-occlusion templates.

**Needs** — [`dx11HDAOCSBlender.h`](dx11HDAOCSBlender.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11HDAOCSBlender.h`](dx11HDAOCSBlender.h.md)
**Tier floor** — T2: two pass descriptions.

## Purpose

An alternative ambient-occlusion implementation that runs as a *compute* dispatch rather than a rasterized full-screen pass. It is offered where the device has compute shaders, alongside the rasterized variants in [`blender_ssao.cpp`](blender_ssao.cpp.md); the console chooses between them.

## `CBlender_CS_HDAO.compile(context)` / `CBlender_CS_HDAO_MSAA.compile(context)`

**Contract** — each emits a single compute pass naming one program (`ssao_hdao` or `ssao_hdao_msaa`), bound to the g-buffer position and normal targets, point-sampled. No render state at all: a compute dispatch has no rasterizer, no blend and no depth.

```text
FUNCTION compile(C)
  base_compile(C)
  IF C.element != 0 THEN RETURN
  C.compute_pass("ssao_hdao" or "ssao_hdao_msaa")
  C.texture("s_position", g-buffer position)
  C.texture("s_normal",   g-buffer normal)
  C.sampler(point, no filtering)
  C.end()
```

**Invariants** — the output surface is *not* bound here. A compute pass writes through an unordered-access binding that the draw code sets up, because the target depends on which resolution the occlusion is being computed at. That is a real asymmetry with the rasterized templates, where the target is the render target and the material need not mention it.

**Notes** — Unlike every other pair in this directory, the multisample variant here carries no sample index: the compute program iterates the samples itself. That is why it is a separate class with its own program name rather than the name/definition mechanism the accumulation blenders use.
