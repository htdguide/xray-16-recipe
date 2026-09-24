# src/xrGame/ui/UIVotingCategory.h

> Declares the vote-starting menu: seven subjects, each enabled or disabled by a server-supplied
> mask.

**Needs** — [`UIVotingCategory.cpp`](UIVotingCategory.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`game_cl_mp.cpp`](../game_cl_mp.cpp.md) · [`UIVotingCategory.cpp`](UIVotingCategory.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIVotingCategory.cpp`](UIVotingCategory.cpp.md). The menu owns
the four follow-up dialogs it can open — kick/ban, change map, change weather, change game type —
constructing each lazily and keeping it for the rest of the session.

Exported units:

- `CUIVotingCategory` — the menu.
- `OnBtn(index)` — take the numbered subject.
- `OnBtnCancel()`.
