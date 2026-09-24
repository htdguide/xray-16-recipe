# src/xrGame/ui/UIMessagesWindow.h

> Declares the message overlay: the PDA message log in single player, and the chat window plus
> two separate logs in multiplayer.

**Needs** — [`UIMessagesWindow.cpp`](UIMessagesWindow.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIGameCustom.cpp`](../UIGameCustom.cpp.md) · [`game_cl_base.cpp`](../game_cl_base.cpp.md) · [`game_cl_capture_the_artefact_messages_menu.cpp`](../game_cl_capture_the_artefact_messages_menu.cpp.md) · [`game_cl_mp.cpp`](../game_cl_mp.cpp.md) · [`game_cl_mp_messages_menu.cpp`](../game_cl_mp_messages_menu.cpp.md) · [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md) · [`UIMessagesWindow.cpp`](UIMessagesWindow.cpp.md) · [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMessagesWindow.cpp`](UIMessagesWindow.cpp.md).

## Exported units

- **The message overlay** — a full-canvas window holding one, two or three logs depending on
  the game kind.
- `AddIconedPdaMessage` — append a news record as a laid-out, self-fading row.
- `AddLogMessage` in two forms — a plain line, and a structured kill report.
- `AddChatMessage` — append a chat line with an author.
- `PendingMode` — switch the chat log between its two authored rectangles.
- `GetChatWnd` — the chat entry widget, for whoever owns keyboard focus.
- `Show` — propagate visibility to the children that exist.
