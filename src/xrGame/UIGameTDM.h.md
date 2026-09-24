# src/xrGame/UIGameTDM.h

> Declares the team-deathmatch game UI implemented in [`UIGameTDM.cpp`](UIGameTDM.cpp.md).

**Needs** — [`UIGameDM.h`](UIGameDM.h.md) · [`ui/UISpawnWnd.h`](ui/UISpawnWnd.h.md)
**Used by** — [`UIGameAHunt.cpp`](UIGameAHunt.cpp.md) · [`UIGameAHunt.h`](UIGameAHunt.h.md) · [`UIGameTDM.cpp`](UIGameTDM.cpp.md) · [`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md) · [`game_cl_teamdeathmatch.h`](game_cl_teamdeathmatch.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CUIGameTDM`, the two-team specialization of the deathmatch game UI. Substance
is in [`UIGameTDM.cpp`](UIGameTDM.cpp.md).

Exported units:

- `CUIGameTDM` — adds per-team icons and scores, the team scoreboard and the team-select
  window to the deathmatch layer.
- `m_pUITeamSelectWnd` — the team-choice window, public because the game mode shows it.
- `SetScoreCaption`, `SetFraglimit`, `SetBuyMsgCaption` — the three writes the game mode
  makes into this layer.
- `Init(stage)` — the three-stage construction described in the twin.
