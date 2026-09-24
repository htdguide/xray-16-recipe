# src/xrGame/ui/UISleepStatic.h

> Declares the sleep-screen widget that paints the hours a nap will cover as a window onto a
> 24-hour strip texture.

**Needs** — [`UISleepStatic.cpp`](UISleepStatic.cpp.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`ScriptXMLInit.cpp`](../ScriptXMLInit.cpp.md) · [`UISleepStatic.cpp`](UISleepStatic.cpp.md) · [`UIXmlInit.cpp`](UIXmlInit.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UISleepStatic.cpp`](UISleepStatic.cpp.md). The type is
exported to scripts as a subclass of the plain picture widget, which is how the sleep screen — a
script — puts one on screen.

Exported units:

- `CUISleepStatic` — the strip-window widget.
- `Draw` / `Update` — paint the (up to two) pieces, recompute them from the clock.
- `InitTextureEx(texture, material)` — bind the strip texture to both pieces.
