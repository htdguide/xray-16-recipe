# src/xrGame/UIPanelsClassFactory.h

> Declares the scoreboard panel factory implemented in [`UIPanelsClassFactory.cpp`](UIPanelsClassFactory.cpp.md).

**Needs** — [`UITeamState.h`](UITeamState.h.md) · [`UIPlayerItem.h`](UIPlayerItem.h.md)
**Used by** — [`UIPanelsClassFactory.cpp`](UIPanelsClassFactory.cpp.md) · [`UITeamPanels.cpp`](UITeamPanels.cpp.md) · [`UITeamPanels.h`](UITeamPanels.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `UIPanelsClassFactory`. Substance is in
[`UIPanelsClassFactory.cpp`](UIPanelsClassFactory.cpp.md).

Exported units:

- `UIPanelsClassFactory` — a stateless factory, held by value inside the panel container.
- `CreateTeamPanel(name, container)` — authored panel name to a team-bound panel.
