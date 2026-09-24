# src/xrGame/ui/UIStatsPlayerList.h

> Declares the scoreboard list for one team: the column set, the three text styles, the header rows,
> and the pooled rows.

**Needs** — [`UIStatsPlayerList.cpp`](UIStatsPlayerList.cpp.md) · [`UIStatsPlayerInfo.h`](UIStatsPlayerInfo.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md)
**Used by** — [`UIStats.cpp`](UIStats.cpp.md) · [`UIStats.h`](UIStats.h.md) · [`UIStatsPlayerList.cpp`](UIStatsPlayerList.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIStatsPlayerList.cpp`](UIStatsPlayerList.cpp.md).

Exported units:

- `CUIStatsPlayerList` — the list: owns the column set, builds two header rows, and keeps the row
  count in step with the player count.
- `Init(document, path)` — read the columns, the styles, and the headers.
- `SetTeam(team)` / `SetSpectator(flag)` — which players this list admits.
- `GetHeader()` / `GetTeamHeader()` — the two header widgets, for the caller to place.
- `AddField(name, width)` — append a column.
- `AddWindow(...)` — **overridden to do nothing**; rows are added only by the list itself.
