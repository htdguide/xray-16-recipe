# src/Layers/xrRender/dxThunderboltRender.h

> Declares the renderer-side filling of the thunderbolt interface.

**Needs** — [`Include/xrRender/ThunderboltRender.h`](../../Include/xrRender/ThunderboltRender.h.md) · [`dxThunderboltRender.cpp`](dxThunderboltRender.cpp.md)
**Used by** — [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md) · [`dxThunderboltRender.cpp`](dxThunderboltRender.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxThunderboltRender.cpp`](dxThunderboltRender.cpp.md): the concrete bolt renderer satisfying [`IThunderboltRender`](../../Include/xrRender/ThunderboltRender.h.md).

Exported units:

- **`dxThunderboltRender`** — holds the bolt mesh's geometry declaration and the glow quad's, builds them at construction and releases them at teardown, and implements `copy` and `render`. Construction touches the device, so an instance may only exist while a device does — which is why the factory, not a static, owns its lifetime.
