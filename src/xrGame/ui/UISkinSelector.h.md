# src/xrGame/ui/UISkinSelector.h

> Declares the multiplayer skin-picking screen: a paged strip of character pictures with keyboard
> shortcuts.

**Needs** — [`UISkinSelector.cpp`](UISkinSelector.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`UIGameCTA.cpp`](../UIGameCTA.cpp.md) · [`game_cl_artefacthunt.cpp`](../game_cl_artefacthunt.cpp.md) · [`game_cl_capture_the_artefact.cpp`](../game_cl_capture_the_artefact.cpp.md) · [`game_cl_deathmatch.cpp`](../game_cl_deathmatch.cpp.md) · [`game_cl_teamdeathmatch.cpp`](../game_cl_teamdeathmatch.cpp.md) · [`UISkinSelector.cpp`](UISkinSelector.cpp.md)
**Tier floor** — T3: a screen declaration

## Purpose

Declares the surface implemented in [`UISkinSelector.cpp`](UISkinSelector.cpp.md), plus the
enumeration naming the screen's three optional buttons — back, spectator, auto-select — which the
multiplayer game state shows or hides per game mode.

Exported units:

- `CUISkinSelectorWnd(section, team)` — the screen, built from a configuration section naming the
  available skins and the team it is picking for.
- `Init(section)` — build from the layout document.
- `SetVisibleForBtn(which, visible)` / `SetCurSkin(index)` / `GetActiveIndex()` / `GetTeam()`.
