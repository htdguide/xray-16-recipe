# src/xrGame/ui/UIVote.h

> Declares the vote-in-progress screen: the subject, three lists of players by how they voted, and
> three buttons.

**Needs** — [`UIVote.cpp`](UIVote.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`game_cl_capture_the_artefact.cpp`](../game_cl_capture_the_artefact.cpp.md) · [`game_cl_deathmatch.cpp`](../game_cl_deathmatch.cpp.md) · [`UIVote.cpp`](UIVote.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIVote.cpp`](UIVote.cpp.md).

Exported units:

- `CUIVote` — the screen.
- `SetVoting(text)` — set the subject line.
- `OnBtnYes` / `OnBtnNo` / `OnBtnCancel`.
