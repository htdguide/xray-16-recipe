# src/Layers/xrRenderDX11/StateManager/dx11ShaderResourceStateCache.h

> Declares the per-stage bound-resource shadow and its dirty range.

**Needs** — [`dx11ShaderResourceStateCache.cpp`](dx11ShaderResourceStateCache.cpp.md) · [`../dx11SH_Texture.cpp`](../dx11SH_Texture.cpp.md)
**Used by** — [`dx11ShaderResourceStateCache.cpp`](dx11ShaderResourceStateCache.cpp.md) · [`dx11R_Backend_Runtime.h`](../dx11R_Backend_Runtime.h.md) · [`dx11SH_Texture.cpp`](../dx11SH_Texture.cpp.md)
**Tier floor** — T1: fixed-size arrays sized by device slot limits.

## Purpose

Declares the surface implemented in [`dx11ShaderResourceStateCache.cpp`](dx11ShaderResourceStateCache.cpp.md). One of these belongs to each command list, because two command lists recording in parallel have independent bindings.

## Exported units

- **six per-stage setters** — record a resource in a slot.
- **`Apply`** — flush the dirty ranges to the given submission context.
- **`ResetDeviceState`** — forget everything; use after the device's bindings change outside this cache.

**Notes** — The slot capacity differs per stage and is taken from the texture layer's limits, not from a single number. That is deliberate: the engine binds many textures to the pixel stage, a few to the vertex stage (for displacement and detail lookup) and one or two elsewhere, and sizing each array to its real need keeps the flush loop's working set small.
