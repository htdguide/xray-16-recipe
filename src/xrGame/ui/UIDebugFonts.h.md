# src/xrGame/ui/UIDebugFonts.h

> Declares the font-sheet inspection screen.

**Needs** — [`UIDebugFonts.cpp`](UIDebugFonts.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIDebugFonts.cpp`](UIDebugFonts.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIDebugFonts.cpp`](UIDebugFonts.cpp.md).

## Exported units

- **`CUIDebugFonts`** — a dialog screen listing every built font.
  - `FillUpList` — rebuild the rows from the font manager's current list; public so the
    screen can be refreshed after a language or resolution change rebuilds the fonts.
  - keyboard handling — quit closes, screenshot falls through.
  - a background picture and a debug type name for the inspector.
