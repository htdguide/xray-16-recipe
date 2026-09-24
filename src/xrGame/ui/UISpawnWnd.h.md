# src/xrGame/ui/UISpawnWnd.h

> Declares the two-team selection screen shown before spawning in a team multiplayer round.

**Needs** — [`UISpawnWnd.cpp`](UISpawnWnd.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`UIGameAHunt.h`](../UIGameAHunt.h.md) · [`UIGameCTA.cpp`](../UIGameCTA.cpp.md) · [`UIGameTDM.h`](../UIGameTDM.h.md) · [`UISpawnWnd.cpp`](UISpawnWnd.cpp.md)
**Tier floor** — T3: a screen declaration

## Purpose

Declares the surface implemented in [`UISpawnWnd.cpp`](UISpawnWnd.cpp.md), plus the enumeration
naming the screen's three optional buttons — back, spectator, auto-select — which the multiplayer
game state shows or hides per game mode. It is the team-picking twin of
[`UISkinSelector`](UISkinSelector.cpp.md), and the two share their button vocabulary and their
scoreboard-while-open behaviour without sharing any code.

Exported units:

- `CUISpawnWnd` — the screen: two team pictures and three buttons.
- `SetVisibleForBtn(which, visible)` / `SetCurTeam(team)`.
