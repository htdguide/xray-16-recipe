# src/Layers/xrRender/dxThunderboltDescRender.h

> Declares the renderer-side filling of the thunderbolt-description interface.

**Needs** — [`Include/xrRender/ThunderboltDescRender.h`](../../Include/xrRender/ThunderboltDescRender.h.md) · [`Include/xrRender/RenderDetailModel.h`](../../Include/xrRender/RenderDetailModel.h.md) · [`dxThunderboltDescRender.cpp`](dxThunderboltDescRender.cpp.md)
**Used by** — [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md) · [`dxThunderboltDescRender.cpp`](dxThunderboltDescRender.cpp.md) · [`dxThunderboltRender.cpp`](dxThunderboltRender.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxThunderboltDescRender.cpp`](dxThunderboltDescRender.cpp.md): the concrete descriptor companion satisfying [`IThunderboltDescRender`](../../Include/xrRender/ThunderboltDescRender.h.md).

Exported units:

- **`dxThunderboltDescRender`** — implements `copy`, `create_model` and `destroy_model`, and exposes the loaded mesh as a **readable field**. The field is deliberately public: [`dxThunderboltRender`](dxThunderboltRender.cpp.md) reaches into it for the mesh's vertex/index counts and material every time a bolt is drawn, and routing that through accessors would buy nothing. A rebuild may make it a read-only property, but the coupling between the two companions is real and should stay visible.
