# src/Layers/xrRender/FProgressive.h

> Declares the continuously-simplifying static model: one vertex and index buffer holding every level of detail at once, addressed by a sliding window.

**Needs** — [`FVisual.h`](FVisual.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md)
**Used by** — [`FProgressive.cpp`](FProgressive.cpp.md) · [`FSkinned.h`](FSkinned.h.md) · [`ModelPool.cpp`](ModelPool.cpp.md)
**Tier floor** — T1: it is a static model plus an index-range table.

## Purpose

Declares the surface implemented in [`FProgressive.cpp`](FProgressive.cpp.md).

## Exported units

- **`ProgressiveVisual`** — a static model carrying a *slide-window table*: one entry per detail level, each naming a vertex count, an index offset and a triangle count. Plus a second such table for the fast geometry, and the last detail level chosen, remembered for the case where the caller does not supply one.
