# src/Layers/xrRender/xrStripify.h

> Declares the two-call vertex-cache optimization surface: reorder a mesh, and score an index order.

**Needs** — [`xrStripify.cpp`](xrStripify.cpp.md)
**Used by** — [`DetailModel.cpp`](DetailModel.cpp.md) · [`xrStripify.cpp`](xrStripify.cpp.md)
**Tier floor** — T3: two declarations.

## Purpose

Declares the surface implemented in [`xrStripify.cpp`](xrStripify.cpp.md), which is where
the output guarantees and the accept/reject rule live. The header exists so that the rest
of the renderer can reorder a mesh without seeing the vendored stripifier's own header.

## Exported units

- **`xrStripify`** — reorder an index list in place for a given transform cache size, and
  hand back the vertex permutation the caller must apply.
- **`xrSimulate`** — count the vertex transforms an index order would cost at a given cache
  size. The measurement that decides whether a reorder is kept.
