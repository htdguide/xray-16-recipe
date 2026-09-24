# src/Layers/xrRender/dxDebugRender.h

> Declares the renderer-side filling of the debug-drawing port.

**Needs** — [`Include/xrRender/DebugRender.h`](../../Include/xrRender/DebugRender.h.md) · [`dxDebugRender.cpp`](dxDebugRender.cpp.md)
**Used by** — [`dxDebugRender.cpp`](dxDebugRender.cpp.md) · [`xrRender_R4.cpp`](../xrRenderPC_R4/xrRender_R4.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxDebugRender.cpp`](dxDebugRender.cpp.md): the concrete debug renderer that satisfies [`IDebugRender`](../../Include/xrRender/DebugRender.h.md). The whole declaration exists only in debug builds.

Exported units:

- **`dxDebugRender`** — holds the line accumulator (vertices, indices, and the two 32767 limits that bound it) and the lazily created table of named debug materials; implements the accumulate-and-flush pair, the calls routed onto the draw stream, the named-material pair, and the immediate triangle draw.
- **`DebugRenderImpl`** — the single shared instance the engine draws through.
- **`rdebug_render`** — a pointer to a second instance that registers with the frame loop and draws during the render phase instead; see the notes in the implementation twin.
