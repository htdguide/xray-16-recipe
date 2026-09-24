# src/Layers/xrRender/R_DStreams.h

> Declares the two per-frame scratch geometry rings.

**Needs** — [`R_DStreams.cpp`](R_DStreams.cpp.md) · [`BufferUtils.h`](BufferUtils.h.md)
**Used by** — [`D3DUtils.cpp`](D3DUtils.cpp.md) · [`D3DUtils.h`](D3DUtils.h.md) · [`D3DXRenderBase.h`](D3DXRenderBase.h.md) · [`DetailManager_soft.cpp`](DetailManager_soft.cpp.md) · [`FLOD.cpp`](FLOD.cpp.md) · [`ParticleEffect.cpp`](ParticleEffect.cpp.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_DBG.cpp`](R_Backend_DBG.cpp.md) · [`R_DStreams.cpp`](R_DStreams.cpp.md) · [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md) · [`SkeletonX.cpp`](SkeletonX.cpp.md) · [`SkeletonXVertRender.h`](SkeletonXVertRender.h.md) · [`dxRainRender.cpp`](dxRainRender.cpp.md) · [`r__dsgraph_render_lods.cpp`](r__dsgraph_render_lods.cpp.md) · _and 1 more_
**Tier floor** — T1: it declares a byte cursor over a device buffer.

## Purpose

Declares the surface implemented in [`R_DStreams.cpp`](R_DStreams.cpp.md). Two near-identical types rather than one parameterized one, because the vertex ring carries a caller-supplied stride at every call and the index ring does not.

Exported units:

- the vertex ring — `Create`, `Destroy`, `Lock`(count, stride) → (pointer, first vertex), `Unlock`(count, stride), `Flush`, `Buffer`, `DiscardID`, `GetSize`, `reset_begin` / `reset_end`, and the publicly readable stashed pre-reset buffer handle;
- the index ring — the same set with `Lock`(count) → (pointer, first index) and `Unlock`(count), and its own stashed handle.

**Notes** — `DiscardID` is public and read by every caller that caches a batch across frames; it is the published half of the ring's contract, not an internal counter. The stashed handles are public for the same reason: the resource manager's reset path reads them directly.
