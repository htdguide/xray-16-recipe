# src/Include/xrRender/FontRender.h

> The renderer's half of a font: its texture atlas and the draw of one frame's worth of accumulated text.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`dxFontRender.cpp`](../../Layers/xrRender/dxFontRender.cpp.md) · [`dxFontRender.h`](../../Layers/xrRender/dxFontRender.h.md) · [`GameFont.h`](../../xrEngine/GameFont.h.md)
**Tier floor** — T2: two calls; the vertex assembly is on the implementor's side.

## Purpose

The engine owns a font: its glyph metrics, its kerning, its size, and a queue of strings to draw with positions and colours, accumulated over the frame. The renderer owns the atlas texture and the material that samples it, and turns the queue into geometry.

One instance per loaded font, created and destroyed by the font through the [render factory](RenderFactory.h.md).

## State

```text
RECORD FontRenderState
  material : Material      # the font's pass chain bound to its atlas texture
```

## `IFontRender`

### `initialize(material_name, texture_name)`

**Contract** — resolves the font's material and atlas texture. Called when the font's definition is read, and again after a device reset. The engine passes both names from the font's configuration.

### `render(font)`

**Contract** — draws everything the font accumulated this frame, then the caller clears the queue. The implementor reads the queued strings, the glyph metrics and the font's current transform from the font object it is handed — which is why the concrete renderer type is a friend of the font: **the geometry is built from the engine's font state, not from anything passed across this interface.**

**Notes** — That reaching-back is the file's one real decision, and it is a deliberate trade. The alternative is a per-glyph or per-string push across the interface, which would put one virtual call in the innermost loop of text rendering; the engine draws a great deal of text (the whole heads-up display, every debug statistic). So the interface stays two calls wide and the implementor is given privileged read access to the font instead.

A rebuild can have this both ways by handing the implementor the queue as a data buffer — a span of (glyph, position, colour) records the engine already has — which keeps the call count at one per font per frame without the privileged access. That is the shape to aim for.

Text is drawn late, after the world and most of the UI, in screen space, with the font's atlas as the only texture.
