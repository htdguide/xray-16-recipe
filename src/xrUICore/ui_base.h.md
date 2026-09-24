# src/xrUICore/ui_base.h

> Declares the module-wide runtime object implemented in [`ui_base.cpp`](ui_base.cpp.md), and the two global reach-throughs the rest of the codebase uses to find it.

**Needs** — [`ui_base.cpp`](ui_base.cpp.md) · [`ui_defs.h`](ui_defs.h.md) · [`ui_focus.h`](ui_focus.h.md) · [`ui_debug.h`](ui_debug.h.md) · [`FontManager/FontManager.h`](FontManager/FontManager.h.md) · [`xrEngine/device.h`](../xrEngine/device.h.md)
**Used by** — [`DBG_Car.cpp`](../xrGame/DBG_Car.cpp.md) · [`HUDCrosshair.cpp`](../xrGame/HUDCrosshair.cpp.md) · [`MainMenu.h`](../xrGame/MainMenu.h.md) · [`debug_text_tree.cpp`](../xrGame/debug_text_tree.cpp.md) · [`file_transfer.cpp`](../xrGame/file_transfer.cpp.md) · [`level_debug.cpp`](../xrGame/level_debug.cpp.md) · [`player_hud.cpp`](../xrGame/player_hud.cpp.md) · [`player_hud_tune.cpp`](../xrGame/player_hud_tune.cpp.md) · [`UICursor.cpp`](Cursor/UICursor.cpp.md) · [`UILine.cpp`](Lines/UILine.cpp.md) · [`UILines.cpp`](Lines/UILines.cpp.md) · [`UISubLine.cpp`](Lines/UISubLine.cpp.md) · [`UIListWnd.cpp`](ListWnd/UIListWnd.cpp.md) · [`UIProgressBar.cpp`](ProgressBar/UIProgressBar.cpp.md) · _and 13 more_
**Tier floor** — T3: a declaration of one object's surface.

## Purpose

Declares the type whose substance lives in [`ui_base.cpp`](ui_base.cpp.md). One decision is
made only here: the object **subscribes to two reset notifications**, one from the graphics
device and one from the UI itself. They are different events. A device reset means the
resolution changed and the scale must be recomputed; a UI reset means the style or the data
changed and everything loaded from data must be thrown away and reloaded. Conflating them
would reload every texture on an alt-tab.

The other decision visible here is that the module's four services — fonts, cursor,
navigation focus, debug overlay — are *members* of this object rather than independent
globals. There is exactly one place to look for them and one lifetime that governs them all.

## Exported units

- `UICore()` / `~UICore()` — construct and release the module's services
- `Font()` / `GetUICursor()` / `Focus()` / `Debugger()` — the four services
- `ReadTextureInfo()` — load the shared texture registry from data
- `ClientToScreenScaledX/Y`, `ClientToScreenScaled` (point and in-place forms),
  `ClientToScreenScaledWidth/Height`, `AlignPixel` — canvas/pixel conversions
- `ScreenFrustum()` / `ScreenFrustumLIT()` — the clip regions for screen-space and
  world-space UI
- `PushScissor(rect, overlapped)` / `PopScissor()` — the clip stack
- `pp_start()` / `pp_stop()` — the post-process drawing bracket
- `RenderFont()` — flush the frame's batched text
- `OnDeviceReset()` / `OnUIReset()` — the two notification handlers
- `is_widescreen()` / `get_current_kx()` — display aspect queries
- `get_xml_name(path, name)` — resolve a layout document against the display aspect
- `m_Scissors` — the clip stack, public because a few callers inspect the top directly
- `m_currentPointType` — screen-space or world-space; read by every draw path

Free functions:

- `UI()` — the module object
- `GetUICursor()` — its cursor, as a shorthand
