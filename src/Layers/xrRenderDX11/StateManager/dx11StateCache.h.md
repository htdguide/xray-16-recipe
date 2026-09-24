# src/Layers/xrRenderDX11/StateManager/dx11StateCache.h

> Declares the three interning caches — rasterizer, depth-stencil and blend — and the record they keep.

**Needs** — [`dx11StateCacheImpl.h`](dx11StateCacheImpl.h.md) · [`dx11StateCache.cpp`](dx11StateCache.cpp.md)
**Used by** — [`dx11State.cpp`](dx11State.cpp.md) · [`dx11StateCache.cpp`](dx11StateCache.cpp.md) · [`dx11StateCacheImpl.h`](dx11StateCacheImpl.h.md) · [`dx11StateManager.cpp`](dx11StateManager.cpp.md) · [`dx11StateManager.h`](dx11StateManager.h.md)
**Tier floor** — T1: it owns device objects and must release them on a defined event.

## Purpose

Declares one cache type parameterized by (device object kind, description kind), and the three process-wide instances of it. The algorithm is in [`dx11StateCacheImpl.h`](dx11StateCacheImpl.h.md); the per-kind creation call is in [`dx11StateCache.cpp`](dx11StateCache.cpp.md).

## Exported units

- **the cache** — `GetState` from a recorded material block or from a description; `ClearStateArray` to release everything.
- **the three instances** — rasterizer states, depth-stencil states, blend states. Each is a process-wide singleton, not per command list, because the objects are immutable and shareable across every context.

**Notes** — Two lifetime rules travel with this type and a rebuild must honour both. First, **cached state objects live for the life of the device, never individually released**, which is what allows every pass state to hold non-owning references to them. Second, `ClearStateArray` may be called *only* at device teardown, because the pass states that point into the cache have no way of learning that their target died — they must all be rebuilt anyway.
