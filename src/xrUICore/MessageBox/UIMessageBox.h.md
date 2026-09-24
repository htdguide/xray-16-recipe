# src/xrUICore/MessageBox/UIMessageBox.h

> Declares the modal dialog — a static whose child set is chosen by a style named in the data, and which reports the player's answer as a distinct message per button.

**Needs** — [`UIMessageBox.cpp`](UIMessageBox.cpp.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`Buttons/UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`EditBox/UIEditBox.h`](../EditBox/UIEditBox.h.md)
**Used by** — [`UIMPAdminMenu.cpp`](../../xrGame/ui/UIMPAdminMenu.cpp.md) · [`UIMPAdminMenu.h`](../../xrGame/ui/UIMPAdminMenu.h.md) · [`UIMessageBoxEx.cpp`](../../xrGame/ui/UIMessageBoxEx.cpp.md) · [`UIMessageBoxEx.h`](../../xrGame/ui/UIMessageBoxEx.h.md) · [`UIScriptWnd_script.cpp`](../../xrGame/ui/UIScriptWnd_script.cpp.md) · [`UIMessageBox.cpp`](UIMessageBox.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMessageBox.cpp`](UIMessageBox.cpp.md).

The load-bearing decision is that a message box has *no* layout of its own. It is named by a
template identifier; a shared document holds one element per template; the template's `type`
attribute selects which children exist, and the children's own elements give their geometry.
So adding a dialog to the game is a data change, and the ten styles below are a **frozen**
vocabulary.

## Exported units

- `CUIMessageBox` — the dialog.
- `E_MESSAGEBOX_STYLE` — the ten styles, named in data as `ok`, `info`, `yes_no`,
  `yes_no_cancel`, `yes_no_copy`, `direct_ip`, `password`, `ra_login`, `quit_windows`,
  `quit_game`.
- `InitMessageBox(template_name)` — build from the template; reports failure when the template
  is absent.
- `Clear` — release every child.
- `SetText` / `GetText` — the message body.
- `GetHost` / `GetPassword` / `GetUserPassword` / `GetTextEditURL` / `SetTextEditURL` — the
  entry fields the networking styles add.
- `SetPasswordMode` / `SetUserPasswordMode` — show or hide one of the two password rows.
- `OnYesOk` — the affirmative path, exposed so a caller can trigger it without a click.

**Notes** — the styles that exist only for the multiplayer server browser (`direct_ip`,
`password`, `ra_login`) reach the dead matchmaking seam. A rebuild may keep the dialog and
retire the callers.
