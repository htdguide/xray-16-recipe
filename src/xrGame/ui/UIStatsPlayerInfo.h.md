# src/xrGame/ui/UIStatsPlayerInfo.h

> Declares one scoreboard row — a strip of cells whose names and widths come from the layout — and
> the field description the list hands it.

**Needs** — [`UIStatsPlayerInfo.cpp`](UIStatsPlayerInfo.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIStatsPlayerInfo.cpp`](UIStatsPlayerInfo.cpp.md) · [`UIStatsPlayerList.cpp`](UIStatsPlayerList.cpp.md) · [`UIStatsPlayerList.h`](UIStatsPlayerList.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIStatsPlayerInfo.cpp`](UIStatsPlayerInfo.cpp.md), and the
record describing one column: a field name and a width. The column list is built once by the player
list and **shared by reference** with every row, so a row reads the layout's column set without
copying it.

Exported units:

- `PI_FIELD_INFO` — one column: name, width.
- `CUIStatsPlayerInfo(columns, font, colour)` — one row, over a shared column list.
- `InitPlayerInfo(position, size)` — lay out the cells.
- `SetInfo(player)` — bind the row to a player for the next update.
