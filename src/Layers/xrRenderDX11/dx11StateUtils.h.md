# src/Layers/xrRenderDX11/dx11StateUtils.h

> Declares the state-description vocabulary: conversions, defaults, normalization, comparison and hashing.

**Needs** — [`dx11StateUtils.cpp`](dx11StateUtils.cpp.md)
**Used by** — [`dx11SamplerStateCache.cpp`](StateManager/dx11SamplerStateCache.cpp.md) · [`dx11State.cpp`](StateManager/dx11State.cpp.md) · [`dx11StateCacheImpl.h`](StateManager/dx11StateCacheImpl.h.md) · [`dx11StateManager.cpp`](StateManager/dx11StateManager.cpp.md) · [`dx11StateUtils.cpp`](dx11StateUtils.cpp.md)
**Tier floor** — T1: it operates on driver description structures.

## Purpose

Declares the free functions implemented in [`dx11StateUtils.cpp`](dx11StateUtils.cpp.md). They are free functions rather than methods because they are applied to descriptions owned by three different callers — the state caches, the state manager and the pass-state builder — none of which owns the others.

## Exported units

- **conversions** — fill mode, cull mode, comparison function, stencil operation, blend factor, blend equation, texture addressing mode, each from the engine's legacy vocabulary to the device's.
- **`ResetDescription`** — one per description kind: the engine's render-state defaults.
- **equality** — one per description kind; a field-wise comparison, because these structures carry uninitialized padding.
- **`GetHash`** — one per description kind; must cover the same fields the comparison does.
- **`ValidateState`** — one per description kind; normalizes fields that cannot matter, so that equivalent descriptions collapse to one cached object.
