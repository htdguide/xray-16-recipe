# src/xrGame/ui/UIRankIndicator.h

> Declares the multiplayer rank badge: ten authored pictures, of which exactly one is attached
> at a time, selected by team and rank together.

**Needs** — [`UIRankIndicator.cpp`](UIRankIndicator.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIGameAHunt.cpp`](../UIGameAHunt.cpp.md) · [`UIGameCTA.cpp`](../UIGameCTA.cpp.md) · [`UIGameDM.cpp`](../UIGameDM.cpp.md) · [`UIGameTDM.cpp`](../UIGameTDM.cpp.md) · [`UIRankIndicator.cpp`](UIRankIndicator.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIRankIndicator.cpp`](UIRankIndicator.cpp.md).

## Exported units

- **The rank badge** — a backing picture plus ten alternative rank pictures.
- `InitFromXml` — build from the overlay layout document; declines when the document has no
  rank panel.
- `SetRank` — show the badge for a (team, rank) pair.
