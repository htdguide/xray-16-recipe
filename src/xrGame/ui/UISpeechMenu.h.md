# src/xrGame/ui/UISpeechMenu.h

> Declares the multiplayer quick-chat menu: a numbered list of canned phrases picked by digit key.

**Needs** — [`UISpeechMenu.cpp`](UISpeechMenu.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`game_cl_mp_messages_menu.cpp`](../game_cl_mp_messages_menu.cpp.md) · [`UISpeechMenu.cpp`](UISpeechMenu.cpp.md)
**Tier floor** — T3: a screen declaration

## Purpose

Declares the surface implemented in [`UISpeechMenu.cpp`](UISpeechMenu.cpp.md). Two of its
declarations are behavioural and belong here because they are the screen's whole personality: it
**needs no cursor** and it **does not stop the player moving**. A quick-chat menu is used mid-fight,
so it must not take the mouse away from aiming or root the player in place.

Exported units:

- `CUISpeechMenu(section)` — the menu, built from a configuration section naming the phrases.
- `InitList(section)` — populate the numbered list.
