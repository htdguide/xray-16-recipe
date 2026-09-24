# src/Layers/xrRender/dxRenderFactory.h

> Declares this backend's filling of the renderer-object factory.

**Needs** — [`Include/xrRender/RenderFactory.h`](../../Include/xrRender/RenderFactory.h.md) · [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md)
**Used by** — [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md) · [`xrRender_R4.cpp`](../xrRenderPC_R4/xrRender_R4.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md).

Exported units:

- **`dxRenderFactory`** — implements the chapter-4 factory: one create/destroy pair per companion interface. The pairs are declared by the same name-driven convention the implementation uses, so the declaration list and the definition list cannot drift apart.
- **`RenderFactoryImpl`** — the single process-wide instance, taken by address when the renderer registers itself in the global environment.
