# src/Layers/xrRender/NvTriStripObjects.h

> Declares the stripifier's working records — triangle, edge, strip — and the stripifier object whose passes operate on them.

**Needs** — [`NvTriStripObjects.cpp`](NvTriStripObjects.cpp.md) · [`VertexCache.h`](VertexCache.h.md)
**Used by** — [`NvTriStrip.cpp`](NvTriStrip.cpp.md) · [`NvTriStripObjects.cpp`](NvTriStripObjects.cpp.md)
**Tier floor** — T2: record declarations over index arithmetic, but the edge record is an intrusive two-list node with a reference count, which is a layout decision rather than a data one.

## Purpose

The private vocabulary of the stripifier. It declares the surface implemented in [`NvTriStripObjects.cpp`](NvTriStripObjects.cpp.md), where the records' invariants and every algorithm live. Nothing outside the stripifier includes it except the front door, which needs the record types to free what the search allocated.

Two of the records carry a few short operations inline — whether a triangle is claimed, whether a strip is speculative — but those are tests over the fields, and the fields' meanings are documented with the search that mutates them, not here.

## Exported units

- **`NvFaceInfo`** — one triangle: three vertex indices plus the three mark fields that record which strip, or which experiment's strip, has claimed it.
- **`NvEdgeInfo`** — one undirected edge: its two endpoints, the up-to-two faces on it, and the two intrusive links that thread it into the edge list of each endpoint. Reference-counted at two because it lives in both lists.
- **`NvStripStartInfo`** — a strip seed: the triangle, the edge to grow along, and which way along it. Pulled out as a record because seeds are collected in batches before any of them is grown.
- **`NvStripInfo`** — one built strip: its seed, its triangles in order, its identity, and the flag used by the final ordering pass. Carries the strip-membership tests, the forward/backward join, the wrap-around guard, the marking, and the growth pass.
- **`NvStripifier`** — the search: the whole pipeline entry point, the flattening pass that produces the index stream, and the two public triangle-comparison helpers the front door also uses.
- **`MyVertex` / `MyVector` / `MyFace`** — mesh records inherited from the library this came from. Nothing reads them.
- **The container aliases and the two-value swap helper** — naming conveniences with no decisions in them.
