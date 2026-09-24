# src/xrGame/UITeamState.h

> Declares one team's scoreboard panel, implemented in [`UITeamState.cpp`](UITeamState.cpp.md).

**Needs** — [`game_cl_base.h`](game_cl_base.h.md) · [`game_base.h`](game_base.h.md) · [`Level.h`](Level.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md)
**Used by** — [`UIPanelsClassFactory.cpp`](UIPanelsClassFactory.cpp.md) · [`UIPanelsClassFactory.h`](UIPanelsClassFactory.h.md) · [`UIPlayerItem.cpp`](UIPlayerItem.cpp.md) · [`UITeamHeader.cpp`](UITeamHeader.cpp.md) · [`UITeamPanels.cpp`](UITeamPanels.cpp.md) · [`UITeamState.cpp`](UITeamState.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `UITeamState`, the rows-and-headers panel for one team. Substance is in
[`UITeamState.cpp`](UITeamState.cpp.md).

Exported units:

- `UITeamState(team, container)` — a panel bound to one team.
- `Init(document, team_node_name, index)` — lay out and build the authored scroll panels.
- `AddPlayer` / `RemovePlayer` / `UpdatePlayer` — membership; removal is deferred and
  `UpdatePlayer` answers false for a player who has left this team.
- `SetArtefactsCount(green, blue)` — the mode pushes both, the panel keeps its own.
- `GetFieldValue(name)` / `GetSummaryFrags()` — the aggregates the header renders.
- `Update` / `Draw` — apply deferred removals, redistribute, re-sort, then draw.
