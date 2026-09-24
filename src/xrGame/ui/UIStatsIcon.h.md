# src/xrGame/ui/UIStatsIcon.h

> Declares the scoreboard cell that shows a small icon — rank, artefact carrier, or dead — from a
> process-wide table built once.

**Needs** — [`UIStatsIcon.cpp`](UIStatsIcon.cpp.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIPlayerItem.cpp`](../UIPlayerItem.cpp.md) · [`UITeamPanels.cpp`](../UITeamPanels.cpp.md) · [`UIStatsIcon.cpp`](UIStatsIcon.cpp.md) · [`UIStatsPlayerInfo.cpp`](UIStatsPlayerInfo.cpp.md) · [`UIStatsPlayerList.cpp`](UIStatsPlayerList.cpp.md)
**Tier floor** — T2: declares a shared table of material handles with an explicit release point

## Purpose

Declares the surface implemented in [`UIStatsIcon.cpp`](UIStatsIcon.cpp.md), and names the eight
icons the scoreboard can show: five ranks (declared as six, one unused), the artefact marker and the
death marker — each in a green and a blue variant, one per team.

Exported units:

- `CUIStatsIcon` — an icon cell; stretches its texture to the cell.
- `SetValue(name)` — choose the icon by the string the scoreboard field produced.
- `InitTexInfo()` / `FreeTexInfo()` — build and release the shared table.
