# src/xrGame/ui/UIStats.h

> Declares the scoreboard section for one team: a header, a player list, and optionally a spectator
> list, stacked in a scroll container.

**Needs** — [`UIStats.cpp`](UIStats.cpp.md) · [`UIStatsPlayerList.h`](UIStatsPlayerList.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md)
**Used by** — [`UIFrags.cpp`](UIFrags.cpp.md) · [`UIFrags.h`](UIFrags.h.md) · [`UIFrags2.cpp`](UIFrags2.cpp.md) · [`UIStats.cpp`](UIStats.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIStats.cpp`](UIStats.cpp.md).

Exported units:

- `CUIStats` — a scroll container holding one team's scoreboard sections.
- `InitStats(document, path, team)` — build the sections and return the team header for the caller
  to place elsewhere.
