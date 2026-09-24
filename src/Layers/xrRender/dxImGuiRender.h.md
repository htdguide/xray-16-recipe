# src/Layers/xrRender/dxImGuiRender.h

> Declares the renderer-side filling of the debug-overlay port.

**Needs** — [`Include/xrRender/ImGuiRender.h`](../../Include/xrRender/ImGuiRender.h.md) · [`dxImGuiRender.cpp`](dxImGuiRender.cpp.md)
**Used by** — [`dxImGuiRender.cpp`](dxImGuiRender.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxImGuiRender.cpp`](dxImGuiRender.cpp.md): the concrete overlay renderer that satisfies [`IImGuiRender`](../../Include/xrRender/ImGuiRender.h.md). It holds no state of its own.

Exported units:

- **`dxImGuiRender`** — implements the copy, the per-frame begin and submit pair, device create and destroy, and the reset bracket. The viewport setup is private.
