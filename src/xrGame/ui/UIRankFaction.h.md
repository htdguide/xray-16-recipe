# src/xrGame/ui/UIRankFaction.h

> Declares one faction's row on the PDA ranking page: its place in the standings, its name,
> icon, home region and strength, and a four-segment bar showing how it feels about the player.

**Needs** — [`UIRankFaction.cpp`](UIRankFaction.cpp.md) · [`FactionState.h`](FactionState.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIRankFaction.cpp`](UIRankFaction.cpp.md) · [`UIRankingWnd.cpp`](UIRankingWnd.cpp.md) · [`UIRankingWnd.h`](UIRankingWnd.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIRankFaction.cpp`](UIRankFaction.cpp.md).

## Exported units

- **The faction row** — about twenty widgets over one faction's state record.
- Constructed either anonymously or bound to a faction identifier; only the bound form is
  useful.
- `init_from_xml` — build every widget and align the two label-and-value pairs.
- `update_info` — refresh from the faction's current state, given the row's place in the
  standings.
- `rating` — tint the two movement arrows according to whether that place improved or worsened.
- `get_faction_power` / `get_cur_sn` — the strength the list sorts by, and the place last shown.
