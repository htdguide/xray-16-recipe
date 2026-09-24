# src/xrUICore/Callbacks/UIWndCallback.h

> Declares the mixin that turns a screen into a message target and lets it bind (widget, event) pairs to handlers.

**Needs** — [`UIWndCallback.cpp`](UIWndCallback.cpp.md) · [`callback_info.h`](callback_info.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md)
**Used by** — [`MainMenu.h`](../../xrGame/MainMenu.h.md) · [`UIActorMenu.h`](../../xrGame/ui/UIActorMenu.h.md) · [`UIActorMenuInitialize.cpp`](../../xrGame/ui/UIActorMenuInitialize.cpp.md) · [`UIChatWnd.h`](../../xrGame/ui/UIChatWnd.h.md) · [`UIDemoPlayControl.cpp`](../../xrGame/ui/UIDemoPlayControl.cpp.md) · [`UIDemoPlayControl.h`](../../xrGame/ui/UIDemoPlayControl.h.md) · [`UIDragDropListEx.cpp`](../../xrGame/ui/UIDragDropListEx.cpp.md) · [`UIDragDropListEx.h`](../../xrGame/ui/UIDragDropListEx.h.md) · [`UIFactionWarWnd.h`](../../xrGame/ui/UIFactionWarWnd.h.md) · [`UILogsWnd.h`](../../xrGame/ui/UILogsWnd.h.md) · [`UIMPAdminMenu.h`](../../xrGame/ui/UIMPAdminMenu.h.md) · [`UIMPChangeMapAdm.h`](../../xrGame/ui/UIMPChangeMapAdm.h.md) · [`UIMPPlayersAdm.h`](../../xrGame/ui/UIMPPlayersAdm.h.md) · [`UIMPServerAdm.h`](../../xrGame/ui/UIMPServerAdm.h.md) · _and 14 more_
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIWndCallback.cpp`](UIWndCallback.cpp.md).

This is the second half of the toolkit's notification story. A widget tells its *message
target* that something happened by sending a numbered event; this mixin is what a screen
inherits so that it can route those numbers to code without writing one giant switch.

## Exported units

- `CUIWndCallback` — the mixin.
- `Register(child)` — point a child's message target at this screen. Every widget whose
  events the screen wants must be registered, otherwise the event goes to the child's parent
  instead.
- `AddCallback(widget, event, handler)` — bind by widget identity.
- `AddCallbackStr(widget_name, event, handler)` — bind by widget *name*, so a screen can bind
  to a widget that the XML created and that the screen never held a pointer to.
- `OnEvent(widget, event, data)` — the dispatch the screen's message handler calls.

**Notes** — the handler type is a two-argument function of (widget, opaque data). The opaque
data is whatever the sending widget chose to attach to that event and is documented per
message in [`UIMessages.h`](../UIMessages.h.md), not here.
