# src/xrGame/ui/UIKeyBinding.h

> Declares the controls page: three headers, a frame, and a scrolling table of action rows.

**Needs** — [`UIKeyBinding.cpp`](UIKeyBinding.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`xrUICore/Windows/UIFrameLineWnd.h`](../../xrUICore/Windows/UIFrameLineWnd.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md)
**Used by** — [`ScriptXMLInit.cpp`](../ScriptXMLInit.cpp.md) · [`UIKeyBinding.cpp`](UIKeyBinding.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIKeyBinding.cpp`](UIKeyBinding.cpp.md).

## Exported units

- **`CUIKeyBinding`** — the controls page. It is also a **binding-table change watcher**,
  registered with the input layer for its whole lifetime; that second role is part of its
  contract, not an implementation detail, because the page must survive the table being
  rewritten by the console or by a defaults restore.
  - `InitFromXml` — dress the frame and the three headers, read the keyboard/gamepad mode
    from a layout attribute, and build the table.
  - `OnKeyMapChanged` — re-read every binding cell on the page.
