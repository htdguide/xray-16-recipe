# src/xrGame/ui/UITabButtonMP.h

> Declares the multiplayer tab button: a tab that is never disabled, carries a separate hint label,
> and shifts its text under the cursor.

**Needs** — [`UITabButtonMP.cpp`](UITabButtonMP.cpp.md) · [`xrUICore/TabControl/UITabButton.h`](../../xrUICore/TabControl/UITabButton.h.md)
**Used by** — [`UIMpItemsStoreWnd.cpp`](UIMpItemsStoreWnd.cpp.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UIMpTradeWnd.cpp`](UIMpTradeWnd.cpp.md) · [`UIMpTradeWnd_init.cpp`](UIMpTradeWnd_init.cpp.md) · [`UITabButtonMP.cpp`](UITabButtonMP.cpp.md) · [`UIXmlInit.cpp`](UIXmlInit.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UITabButtonMP.cpp`](UITabButtonMP.cpp.md). Its fields are
public because its layout reader — `InitTabButtonMP` in [`UIXmlInit`](UIXmlInit.cpp.md) — writes
them directly rather than going through setters; the two files are one unit split for build reasons.

Exported units:

- `CUITabButtonMP` — the tab button.
- `IsEnabled()` — **always true**, overriding the base.
- `SetOrientation(vertical)` / `CreateHint()`.
- `m_text_ident_normal`, `m_text_ident_cursor_over` — the two text offsets.
- `m_hint` — the secondary label, drawn by this button rather than by the tree.
