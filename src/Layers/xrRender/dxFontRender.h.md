# src/Layers/xrRender/dxFontRender.h

> Declares the renderer-side filling of the font port.

**Needs** — [`Include/xrRender/FontRender.h`](../../Include/xrRender/FontRender.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md) · [`dxFontRender.cpp`](dxFontRender.cpp.md)
**Used by** — [`dxFontRender.cpp`](dxFontRender.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`dxFontRender.cpp`](dxFontRender.cpp.md): the concrete font renderer that satisfies [`IFontRender`](../../Include/xrRender/FontRender.h.md).

Exported units:

- **`dxFontRender`** — holds the text material and its vertex format; implements creation from a shader and texture name, and the per-frame draw of the font's queued lines. The single-glyph emitter is private and folded into the draw's pseudocode.
