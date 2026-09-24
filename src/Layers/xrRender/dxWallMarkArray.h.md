# src/Layers/xrRender/dxWallMarkArray.h

> Declares the renderer-side filling of the decal-variant-set interface.

**Needs** — [`Include/xrRender/WallMarkArray.h`](../../Include/xrRender/WallMarkArray.h.md) · [`dxWallMarkArray.cpp`](dxWallMarkArray.cpp.md)
**Used by** — [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md) · [`dxWallMarkArray.cpp`](dxWallMarkArray.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxWallMarkArray.cpp`](dxWallMarkArray.cpp.md): the concrete variant set satisfying [`IWallMarkArray`](../../Include/xrRender/WallMarkArray.h.md).

Exported units:

- **`dxWallMarkArray`** — holds the list of resolved decal materials; implements `copy`, `append_mark`, `clear`, `empty` and the opaque-handle pick `generate_wallmark`, plus the module-internal `generate_wallmark_direct` that returns the concrete material reference for the renderer's own decal batcher.
