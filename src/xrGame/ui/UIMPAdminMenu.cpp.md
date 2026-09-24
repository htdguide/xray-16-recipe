# src/xrGame/ui/UIMPAdminMenu.cpp

> The remote-administration screen: three panels behind a tab strip, of which exactly one is attached to the tree at a time, and a password prompt whose answer is issued as a console command.

**Needs** — [`UIMPAdminMenu.h`](UIMPAdminMenu.h.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`UIMPPlayersAdm.h`](UIMPPlayersAdm.h.md) · [`UIMPServerAdm.h`](UIMPServerAdm.h.md) · [`UIMPChangeMapAdm.h`](UIMPChangeMapAdm.h.md) · [`UIMessageBoxEx.h`](UIMessageBoxEx.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/TabControl/UITabControl.h`](../../xrUICore/TabControl/UITabControl.h.md) · [`xrUICore/MessageBox/UIMessageBox.h`](../../xrUICore/MessageBox/UIMessageBox.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`UIMPAdminMenu.h`](UIMPAdminMenu.h.md)
**Tier floor** — T3.

## Purpose

A server operator's console: manage players, change server settings, change the map. It is
the chapter's clearest example of the **tabbed container** pattern, which several screens in
this chapter use and which is worth stating once.

## State

```text
RECORD AdminMenu EXTENDS DialogScreen
  layout        : Document          # kept alive; the panels read it at construction
  tabs          : TabControl
  panels        : players, server, change-map      # all three built, never destroyed
  active        : optional<Panel>
  active_name   : text
  login_prompt  : MessageBox        # asks for a user name and a password
  error_prompt  : MessageBox
  close_button  : Button
```

**Invariants**

- **All three panels exist for the screen's whole life; only one is attached.** Switching
  detaches and hides the previous panel, then attaches and shows the new one. The panels are
  deliberately not self-deleting, because they outlive every attachment.
- The active panel's name is remembered, and switching to the already-active panel is a
  no-op — a tab control re-announcing its selection must not tear the panel down and rebuild
  it.
- The tab identifier strings — `players`, `server`, `change_map` — are the frozen contract
  with the shipped layout: the tab control reports the authored identifier and this screen
  maps it to a panel by exact match.

## Notification routing

**Contract** — the screen handles two notifications itself and **forwards everything else to
the active panel**:

```text
FUNCTION on_notification(sender, message, payload)
  IF message is "tab changed" and sender is the tab strip
    THEN switch to the panel named by the tab's identifier
  ELSE IF message is "button clicked" and sender is the close button
    THEN close the screen
  ELSE forward to the active panel
```

**Notes** — this is the inverse of chapter 15's default, which rebroadcasts a notification
*down* to every enabled child. Here the screen intercepts what is its own and hands the rest
to one child. That is what lets each panel bind its own widgets without every panel receiving
every other panel's events — and it works only because exactly one panel is attached, so
"the active panel" and "the panel whose widget sent this" are the same.

## Input

**Contract** — the quit binding is context-sensitive: if the server panel is showing a
sub-view with its own back button, quit goes *back within that panel*; otherwise it closes
the screen.

**Notes** — one key, two meanings, decided by asking the panel whether it has somewhere to go
back to. A rebuild generalises this into a "can this panel handle back" question asked of
whichever panel is active, rather than naming one panel.

## Logging in

**Contract** — the login prompt is a message box with a user-name and a password field; its
confirmation handler composes a console command from the two and executes it. Failures are
reported through a second, plain message box whose text the caller supplies.

**Notes** — the console again, and here the indirection is doing real work: administration is
a *protocol* over the network connection, and the console command is the engine's one
implementation of it. The screen is a form over a command line. See
[Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport).

The credentials travel as command arguments, which means they appear in the console history.
That is the shipped behaviour and is worth flagging: a rebuild should give the command a path
that does not log its arguments.

Both message boxes are constructed from **named styles** — chapter 15's message box chooses
its child set by a style named in data — so what fields the login prompt has is authored, not
coded.
