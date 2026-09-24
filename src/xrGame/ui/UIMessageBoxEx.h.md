# src/xrGame/ui/UIMessageBoxEx.h

> Declares the modal wrapper that turns the toolkit's message box into a dialog the screen
> stack can push, with two caller-supplied outcome handlers.

**Needs** — [`UIMessageBoxEx.cpp`](UIMessageBoxEx.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md) · [`xrUICore/MessageBox/UIMessageBox.h`](../../xrUICore/MessageBox/UIMessageBox.h.md)
**Used by** — [`MainMenu.cpp`](../MainMenu.cpp.md) · [`UIGameAHunt.cpp`](../UIGameAHunt.cpp.md) · [`UIGameCTA.cpp`](../UIGameCTA.cpp.md) · [`game_cl_artefacthunt.cpp`](../game_cl_artefacthunt.cpp.md) · [`ServerList.cpp`](ServerList.cpp.md) · [`ServerList.h`](ServerList.h.md) · [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIMPAdminMenu.cpp`](UIMPAdminMenu.cpp.md) · [`UIMPAdminMenu.h`](UIMPAdminMenu.h.md) · [`UIMessageBoxEx.cpp`](UIMessageBoxEx.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMessageBoxEx.cpp`](UIMessageBoxEx.cpp.md).

## Exported units

- **The modal message box** — a dialog wrapping exactly one toolkit message box.
- `InitMessageBox` — build from a named style and adopt its geometry; may decline.
- `SetText` / `GetText` — the body text.
- `func_on_ok` / `func_on_no` — the two outcome handlers, assigned directly by the caller.
- `GetHost`, `GetPassword`, `SetTextEditURL` / `GetTextEditURL` — read the styles that carry
  entry fields.
- `NeedCursor` / `NeedCenterCursor` — whether the screen stack should show a pointer.
