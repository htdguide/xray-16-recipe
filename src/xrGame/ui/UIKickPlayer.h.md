# src/xrGame/ui/UIKickPlayer.h

> Declares the kick/ban vote screen.

**Needs** — [`UIKickPlayer.cpp`](UIKickPlayer.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`UIKickPlayer.cpp`](UIKickPlayer.cpp.md) · [`UIVotingCategory.cpp`](UIVotingCategory.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIKickPlayer.cpp`](UIKickPlayer.cpp.md).

## Exported units

- **`CUIKickPlayer`**
  - `InitKick` / `InitBan` — the two modes, each dressing its own header then running the
    shared construction.
  - `Update` — the once-per-second change-detecting refresh.
  - `SendMessage` — list selection is remembered by name; the two buttons dispatch.
  - `OnBtnOk` / `OnBtnCancel` — issue the vote and close, or just close.
  - `OnKeyboardAction` — the quit binding cancels.
