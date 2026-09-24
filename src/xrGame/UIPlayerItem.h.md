# src/xrGame/UIPlayerItem.h

> Declares the scoreboard row implemented in [`UIPlayerItem.cpp`](UIPlayerItem.cpp.md).

**Needs** — [`game_cl_base.h`](game_cl_base.h.md) · [`Level.h`](Level.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md)
**Used by** — [`UIPanelsClassFactory.h`](UIPanelsClassFactory.h.md) · [`UIPlayerItem.cpp`](UIPlayerItem.cpp.md) · [`UITeamState.cpp`](UITeamState.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `UIPlayerItem`, one player's row in the multiplayer scoreboard. Substance is in
[`UIPlayerItem.cpp`](UIPlayerItem.cpp.md).

Exported units:

- `UIPlayerItem(team, client_id, team_panel, container)` — a row bound to one connected
  client, remembering the team it was created for.
- `Init(document, node_name, index)` — build the authored text and icon fields.
- `GetPlayerCheckPoints()` — the sort key the panel orders rows by.
- `Update()` — refresh from replicated state; also detects disconnect and team change.
