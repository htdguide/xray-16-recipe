# src/xrGame/ui/UIChangeMap.h

> Declares the multiplayer vote dialog for changing the level.

**Needs** — [`UIChangeMap.cpp`](UIChangeMap.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`UIChangeMap.cpp`](UIChangeMap.cpp.md) · [`UIVotingCategory.cpp`](UIVotingCategory.cpp.md)
**Tier floor** — T3: a list, a preview and a console command

## Purpose

Declares the surface implemented in [`UIChangeMap.cpp`](UIChangeMap.cpp.md).

## `CUIChangeMap`

A modal dialog listing the levels available for the current game type, with a preview image
and version label for the selected one, and two buttons. Choosing one **starts a vote**; it
does not change the level.

- `InitChangeMap(document)` — build and fill.
- `OnBtnOk` / `OnBtnCancel` / `OnItemSelect` — the three outcomes.
