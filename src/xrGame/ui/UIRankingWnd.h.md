# src/xrGame/ui/UIRankingWnd.h

> Declares the PDA's ranking page: the player's own record, the faction standings, the
> achievements, the top-kills and favourite-weapon trophies, and a script-driven leaderboard.

**Needs** — [`UIRankingWnd.cpp`](UIRankingWnd.cpp.md) · [`UIRankFaction.h`](UIRankFaction.h.md) · [`UIAchievements.h`](UIAchievements.h.md) · [`UIRankingsCoC.h`](UIRankingsCoC.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md) · [`UIRankingWnd.cpp`](UIRankingWnd.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIRankingWnd.cpp`](UIRankingWnd.cpp.md).

## Exported units

- **The ranking page** — one screen aggregating five independent, independently optional panels.
- `Init` — build from the ranking layout document; declines when the document is absent, which
  is how a game without this page gets none.
- `Show`, `Update`, `DrawHint`, `ResetAll` — the lifecycle; the refresh is on a timer, not per
  frame.
- `update_info` — re-read everything from the game and the script layer.
