# src/Layers/xrRender/VertexCache.h

> Declares the post-transform vertex cache model, with the two hot operations inline.

**Needs** — _(none)_
**Used by** — [`NvTriStripObjects.cpp`](NvTriStripObjects.cpp.md) · [`NvTriStripObjects.h`](NvTriStripObjects.h.md) · [`VertexCache.cpp`](VertexCache.cpp.md) · [`xrStripify.cpp`](xrStripify.cpp.md)
**Tier floor** — T3: a declaration only.

## Purpose

Declares the model implemented in [`VertexCache.cpp`](VertexCache.cpp.md). The residency test and the push-and-evict operation are placed here rather than in the implementation because they are the stripifier's innermost loop; that is a packaging decision only.

## Exported units

- **`VertexCache`** — construct with a capacity (default 16), `InCache` to test residency, `AddEntry` to push a vertex and learn which one fell off, `Clear` to empty it, `Copy` to overwrite another cache with this one's contents, and the indexed `At`/`Set` pair used to snapshot and restore state around a speculative branch.
