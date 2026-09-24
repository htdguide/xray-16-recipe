# src/Layers/xrRender/dxUIShader.h

> Declares the renderer-side filling of the opaque UI material handle.

**Needs** — [`Include/xrRender/UIShader.h`](../../Include/xrRender/UIShader.h.md) · [`dxUIShader.cpp`](dxUIShader.cpp.md) · [`Shader.h`](Shader.h.md)
**Used by** — [`dxDebugRender.cpp`](dxDebugRender.cpp.md) · [`dxRenderFactory.cpp`](dxRenderFactory.cpp.md) · [`dxUIRender.cpp`](dxUIRender.cpp.md) · [`dxUIShader.cpp`](dxUIShader.cpp.md) · [`dxWallMarkArray.cpp`](dxWallMarkArray.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxUIShader.cpp`](dxUIShader.cpp.md): the concrete handle satisfying [`IUIShader`](../../Include/xrRender/UIShader.h.md).

Exported units:

- **`dxUIShader`** — holds the shared material reference and the frozen base-colour sampler name; implements `copy`, `create`, `destroy`, `initialised`, equality, and the three texture queries (`base_texture`, `base_texture_resolution`, `debug_overlay_texture_id`).

**Notes** — The declaration grants direct access to the wrapped material to four privileged readers: the UI vertex sink, the debug renderer, the wallmark array and the renderer core. Each of them needs the concrete material, not the opaque handle, and each lives inside this module. That is the interface's real boundary: *outside* the renderer the handle is opaque; *inside* it is a thin wrapper. A rebuild should express this as module-internal visibility rather than a list of exceptions.
