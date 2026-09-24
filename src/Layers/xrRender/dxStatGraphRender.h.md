# src/Layers/xrRender/dxStatGraphRender.h

> Declares the renderer-side filling of the stat-graph interface.

**Needs** — [`Include/xrRender/StatGraphRender.h`](../../Include/xrRender/StatGraphRender.h.md) · [`xrEngine/StatGraph.h`](../../xrEngine/StatGraph.h.md) · [`dxStatGraphRender.cpp`](dxStatGraphRender.cpp.md)
**Used by** — [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md) · [`dxStatGraphRender.cpp`](dxStatGraphRender.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxStatGraphRender.cpp`](dxStatGraphRender.cpp.md): the concrete graph renderer satisfying [`IStatGraphRender`](../../Include/xrRender/StatGraphRender.h.md).

Exported units:

- **`dxStatGraphRender`** — owns the two geometry declarations (quads and lines) and implements `copy`, the device create/destroy pair and `on_render`. Its private emitters — background, bars, curves, bar outlines, markers — are steps of the single render algorithm and are described there.
