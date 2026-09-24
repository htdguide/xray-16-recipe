# src/xrGame/ui/UIMPAdminMenu.h

> Declares the remote-administration screen: a tabbed modal dialog whose three pages let a
> logged-in server administrator drive the running match.

**Needs** — [`UIMPAdminMenu.cpp`](UIMPAdminMenu.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`UIMPPlayersAdm.h`](UIMPPlayersAdm.h.md) · [`UIMPServerAdm.h`](UIMPServerAdm.h.md) · [`UIMPChangeMapAdm.h`](UIMPChangeMapAdm.h.md) · [`UIMessageBoxEx.h`](UIMessageBoxEx.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md) · [`xrUICore/MessageBox/UIMessageBox.h`](../../xrUICore/MessageBox/UIMessageBox.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`game_cl_mp.cpp`](../game_cl_mp.cpp.md) · [`UIMPAdminMenu.cpp`](UIMPAdminMenu.cpp.md)
**Tier floor** — T3: a screen that composes three sub-screens and issues text commands.

## Purpose

Declares the surface implemented in [`UIMPAdminMenu.cpp`](UIMPAdminMenu.cpp.md). The screen
is a modal dialog that owns one layout document, a tab strip, three mutually exclusive
sub-screens, a close button, and two message boxes — one that asks for the administrator
password, one that reports the outcome.

## Exported units

- **The admin menu screen** — a modal dialog that is also a notification handler.
- `Init` — build the sub-screens from the layout document.
- `SetActiveSubdialog` — show exactly one sub-screen, named by its layout section.
- `RemoteAdminLogin` — take the password out of the login box and issue the login command.
- `ShowMessageBox` — surface a login result, styled, with an optional reason string.
- The notification and keyboard hooks that route the tab strip, the close button and the
  cancel key.
