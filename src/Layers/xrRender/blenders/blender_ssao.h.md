# src/Layers/xrRender/blenders/blender_ssao.h

> Declares the ambient-occlusion templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`blender_ssao.cpp`](blender_ssao.cpp.md)
**Used by** — [`blender_ssao.cpp`](blender_ssao.cpp.md) · [`r3_rendertarget_phase_ssao.cpp`](../../xrRender_R2/r3_rendertarget_phase_ssao.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`blender_ssao.cpp`](blender_ssao.cpp.md).

Exported units:

- **`CBlender_SSAO_noMSAA`** — internal template, no class identifier, neither detailable nor lightmappable; emits the occlusion pass and, on the newer generations, the half-resolution depth downsample. On the oldest deferred generation the class is spelled `CBlender_SSAO` and the other name is an alias for it, so that callers need not branch.
- **`CBlender_SSAO_MSAA`** — the occlusion pass compiled for one multisample sample index; exists only on backends that do per-sample shading.

**Notes** — In the original the namespace's closing brace sits inside the conditional branch that declares the second class, so the oldest deferred generation's build of this header leaves the namespace open and relies on whatever includes it next. It is a latent defect, not a convention; a rebuild closes its scope unconditionally.
