# src/xrUICore/FontManager/FontManager.h

> Declares the fixed set of named fonts the whole UI draws with, and the per-frame flush that turns queued glyphs into draw calls.

**Needs** — [`FontManager.cpp`](FontManager.cpp.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UIDebugFonts.cpp`](../../xrGame/ui/UIDebugFonts.cpp.md) · [`FontManager.cpp`](FontManager.cpp.md) · [`ui_base.cpp`](../ui_base.cpp.md) · [`ui_base.h`](../ui_base.h.md)
**Tier floor** — T2: it owns device-backed font objects and must release and rebuild them on a device or UI reset, but it decides nothing about glyph layout.

## Purpose

Declares the surface implemented in [`FontManager.cpp`](FontManager.cpp.md).

The load-bearing fact is the *closed set*: there is no "load a font by name" entry point.
Eleven fonts exist, each is a named field, and the XML vocabulary maps a font name attribute
onto one of them. Data cannot introduce a twelfth. A rebuild may hold them in a map, but the
names must be exactly these because the shipped XML uses them.

## Exported units

- `CFontManager` — the font set.
- The eleven font fields. Two are for the in-world heads-up display (a medium font and a
  device-independent one); the other nine are UI fonts, named after the typeface and pixel
  size the original art was authored at — several carry a `Russian` suffix that is historical
  and does not mean the font is language-specific.
- `m_all_fonts` — the iteration order, which is also the destruction order.
- `InitializeFonts` / `InitializeFont(font, section, flags)` — build or rebuild from
  configuration.
- `GetFontTexName(section)` — picks the right glyph atlas for the current vertical resolution.
- `Render` — flushes every font's queued text for the frame.
- `OnUIReset` — rebuilds every font in place when the UI is reloaded.
