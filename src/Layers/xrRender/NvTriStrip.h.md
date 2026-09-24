# src/Layers/xrRender/NvTriStrip.h

> Declares the stripifier's public surface: the primitive-group record, the four tuning knobs, and the two entry points.

**Needs** — [`NvTriStrip.cpp`](NvTriStrip.cpp.md)
**Used by** — [`NvTriStrip.cpp`](NvTriStrip.cpp.md) · [`xrStripify.cpp`](xrStripify.cpp.md)
**Tier floor** — T2: a record declaration and four setters, but the record carries a raw owned index array with a 16-bit element width that the caller must respect.

## Purpose

The header a caller includes to reorder a mesh. It declares the surface implemented in [`NvTriStrip.cpp`](NvTriStrip.cpp.md), where the contracts and the output shapes live.

It also carries the two named hardware cache sizes — 16 and 24 — as the only documentation anywhere of what the cache-size knob means. They name the two GPU generations that were current when the tool was written; the engine passes the running device's own reported size instead, and the constants survive only as a hint at the expected magnitude.

## Exported units

- **`PrimitiveGroup`** — one drawable run: a topology tag, an index count, and an owned array of 16-bit indices. Releasing the group releases the array.
- **`PrimType`** — the topology tag: list, strip or fan. Fan is declared and never produced.
- **`SetCacheSize`** — the target post-transform vertex cache size, in vertices. Controls how long the generated strips are.
- **`SetStitchStrips`** — whether to join all strips into one with degenerate triangles, or hand back many.
- **`SetMinStripSize`** — the strip length, in triangles, below which a strip is demoted into the leftover triangle list.
- **`SetListsOnly`** — discard the strip topology and hand back one reordered triangle list.
- **`GenerateStrips`** — the reordering pass: triangle list in, primitive groups out.
- **`RemapIndices`** — the renumbering pass: vertices numbered in first-use order, so the vertex buffer is read front-to-back.
