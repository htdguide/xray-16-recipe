# src/xrGame/UITeamPanels.h

> Declares the scoreboard container implemented in [`UITeamPanels.cpp`](UITeamPanels.cpp.md).

**Needs** — [`UIPanelsClassFactory.h`](UIPanelsClassFactory.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md)
**Used by** — [`UIGameAHunt.cpp`](UIGameAHunt.cpp.md) · [`UIGameCTA.cpp`](UIGameCTA.cpp.md) · [`UIGameDM.cpp`](UIGameDM.cpp.md) · [`UIGameTDM.cpp`](UIGameTDM.cpp.md) · [`UIPanelsClassFactory.cpp`](UIPanelsClassFactory.cpp.md) · [`UIPlayerItem.cpp`](UIPlayerItem.cpp.md) · [`UITeamPanels.cpp`](UITeamPanels.cpp.md) · [`UITeamState.cpp`](UITeamState.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `UITeamPanels`, which owns one panel per authored team and the scoreboard's
layout document. Substance is in [`UITeamPanels.cpp`](UITeamPanels.cpp.md).

Exported units:

- `Init(document_name, root_node)` — load, lay out decoration then panels, seed
  membership, apply phase visibility.
- `AddPlayer` / `RemovePlayer` / `UpdatePlayer` — broadcast to every panel; the panels
  filter by team.
- `NeedUpdatePlayers()` / `NeedUpdatePanels()` — set the deferred-refresh flags that a
  row or the game mode raises.
- `SetArtefactsCount(green, blue)` — forwarded to every panel.
- `Update()` — consume the dirty flags, then update children.
