# src/xrGame/ui/UIChatWnd.h

> Declares the multiplayer chat entry line.

**Needs** — [`UIChatWnd.cpp`](UIChatWnd.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`../../xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIChatWnd.cpp`](UIChatWnd.cpp.md) · [`UIMessagesWindow.cpp`](UIMessagesWindow.cpp.md)
**Tier floor** — T3: an edit box and one send

## Purpose

Declares the surface implemented in [`UIChatWnd.cpp`](UIChatWnd.cpp.md).

## `CUIChatWnd`

A one-line chat entry: a prefix static that says who you are talking to, and an edit box.
Raised as a dialog so it takes the keyboard, but it explicitly **needs no cursor** — the
player is typing, not pointing.

- `Init(document)` — build, and remember two sets of rectangles (see the implementation).
- `SetEditBoxPrefix(text)` — set the prefix and reflow the edit box beside it.
- `ChatToAll(bool)` — whether the *next* message goes to everyone or to the team.
- `PendingMode(bool)` — switch between the two remembered layouts.
