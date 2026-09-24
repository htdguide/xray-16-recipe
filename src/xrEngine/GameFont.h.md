# src/xrEngine/GameFont.h

> Declares the bitmap-atlas font.

**Needs** — [`IGameFont.hpp`](IGameFont.hpp.md) · [`Include/xrRender/FontRender.h`](../Include/xrRender/FontRender.h.md)
**Used by** — [`FontRender.h`](../Include/xrRender/FontRender.h.md) · [`D3DUtils.cpp`](../Layers/xrRender/D3DUtils.cpp.md) · [`dxFontRender.cpp`](../Layers/xrRender/dxFontRender.cpp.md) · [`dxFontRender.h`](../Layers/xrRender/dxFontRender.h.md) · [`Device_Initialize.cpp`](Device_Initialize.cpp.md) · [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`FDemoRecord.h`](FDemoRecord.h.md) · [`GameFont.cpp`](GameFont.cpp.md) · [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md) · [`IPerformanceAlert.hpp`](IPerformanceAlert.hpp.md) · [`PerformanceAlert.cpp`](PerformanceAlert.cpp.md) · [`Stats.cpp`](Stats.cpp.md) · [`profiler.cpp`](profiler.cpp.md) · [`xrSheduler.cpp`](xrSheduler.cpp.md) · _and 18 more_
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`GameFont.cpp`](GameFont.cpp.md) — the engine's one filling of the text interface.

## State

```text
RECORD Font
  char_table    : list<vector3>   # per code point: atlas x, atlas y, advance width
  code_points   : int             # 256 for a single-byte font, 65536 for a multibyte one
  glyph_height  : real            # the atlas cell height; uniform across the font
  space_step    : real            # extra advance for characters that need one; = ceil(height/2)
  interval      : vector2         # multipliers on character advance and line height
  queue         : list<QueuedString>
  current       : colour, height, position, alignment
  backend       : FontRenderer    # the renderer's own half
```

```text
RECORD QueuedString
  text      : text (bounded, 1024 bytes)
  x, y      : real (pixels)
  height    : real
  colour    : int (packed RGBA)
  alignment : left | right | centre
```

## Exported units

- **Two constructors** — from a configuration section (which names shader, texture, optional size and interval), or directly from a shader and texture name.
- **`initialize`** — load the character table and hand the shader and texture to the renderer. See the implementation twin; this is the substantive part.
- **Everything in [`IGameFont.hpp`](IGameFont.hpp.md)** — implemented here.
- **`on_render`** — flush and clear the queue.
- **`clear`** — discard the queue without drawing.

**Notes** — The queued string is a fixed 1024-byte buffer, copied by value. That is a deliberate trade: text is queued from callers whose own buffers are about to go out of scope, so the queue must own its text, and a fixed buffer avoids an allocation per line in a path that runs hundreds of times a frame.

**Notes** — The renderer backend is a friend of the font and reads its character table and queue directly. In a rebuild that is a read-only view passed across the renderer interface.
