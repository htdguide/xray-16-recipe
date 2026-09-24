# src/Layers/xrRenderDX11/StateManager/dx11SamplerStateCache.h

> Declares the sampler cache: handles instead of pointers, one bind call per stage, two globally overridden fields.

**Needs** — [`dx11SamplerStateCache.cpp`](dx11SamplerStateCache.cpp.md)
**Used by** — [`dx11SamplerStateCache.cpp`](dx11SamplerStateCache.cpp.md) · [`dx11State.cpp`](dx11State.cpp.md) · [`dx11State.h`](dx11State.h.md) · [`dx11HW.cpp`](../dx11HW.cpp.md)
**Tier floor** — T1: it owns device objects released at device teardown.

## Purpose

Declares the surface implemented in [`dx11SamplerStateCache.cpp`](dx11SamplerStateCache.cpp.md), plus the process-wide instance.

## Exported units

- **the handle type and its sentinel** — an index; the sentinel means an empty slot. A handle list is what a pass state stores per stage.
- **`GetState`** — intern a description, returning a handle.
- **the six per-stage apply calls** — vertex, pixel, geometry, hull, domain, compute. Six entry points rather than one taking a stage, because each binds through a different device entry; a rebuild with a uniform binding call needs one.
- **`SetMaxAnisotropy` / `SetMipLODBias`** — the two global overrides, applied retroactively to every cached sampler.
- **`ClearStateArray`** — release everything; device teardown only.
- **the process-wide instance**.
