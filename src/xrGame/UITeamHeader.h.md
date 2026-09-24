# src/xrGame/UITeamHeader.h

> Declares the team scoreboard header implemented in [`UITeamHeader.cpp`](UITeamHeader.cpp.md).

**Needs** — [`game_cl_base.h`](game_cl_base.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md)
**Used by** — [`UITeamHeader.cpp`](UITeamHeader.cpp.md) · [`UITeamState.cpp`](UITeamState.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `UITeamHeader`, the label-and-aggregate strip above one team's scroll panel.
Substance is in [`UITeamHeader.cpp`](UITeamHeader.cpp.md).

Exported units:

- `UITeamHeader(parent)` — bound for life to the team panel it reads aggregates from.
- `Init(document, path)` — build authored `column` labels and `field` aggregates.
- `Update()` — refresh each aggregate as `caption: value`.
