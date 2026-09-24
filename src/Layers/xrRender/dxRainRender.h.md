# src/Layers/xrRender/dxRainRender.h

> Declares the renderer-side filling of the rain interface.

**Needs** — [`Include/xrRender/RainRender.h`](../../Include/xrRender/RainRender.h.md) · [`dxRainRender.cpp`](dxRainRender.cpp.md)
**Used by** — [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxRainRender.cpp`](dxRainRender.cpp.md): the concrete rain renderer that satisfies [`IRainRender`](../../Include/xrRender/RainRender.h.md).

Exported units:

- **`dxRainRender`** — holds the streak material and its geometry declaration, the splash detail model and its geometry declaration; implements `render`, `drop_bounds` and `copy`.
