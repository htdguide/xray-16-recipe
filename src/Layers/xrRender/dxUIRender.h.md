# src/Layers/xrRender/dxUIRender.h

> Declares the renderer-side filling of the UI vertex-sink interface.

**Needs** — [`Include/xrRender/UIRender.h`](../../Include/xrRender/UIRender.h.md) · [`dxUIRender.cpp`](dxUIRender.cpp.md)
**Used by** — [`D3DXRenderBase.cpp`](D3DXRenderBase.cpp.md) · [`dxUIRender.cpp`](dxUIRender.cpp.md) · [`xrRender_R4.cpp`](../xrRenderPC_R4/xrRender_R4.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxUIRender.cpp`](dxUIRender.cpp.md): the concrete UI renderer satisfying [`IUIRender`](../../Include/xrRender/UIRender.h.md).

Exported units:

- **`dxUIRender`** — holds the two geometry declarations (screen-space and world-space), the open batch's topology, point kind, reserved count, stream offset and write cursor; implements geometry create/destroy, material and alpha-reference selection, scissoring, the batch start/push/flush triple, the video-material name substitution, and the world-transform and cull-mode passthroughs.
- **`UIRenderImpl`** — the single process-wide instance, taken by address when the renderer registers itself in the global environment.
