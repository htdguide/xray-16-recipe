# src/xrGame/UIGameDM.h

> Declares the deathmatch interface implemented in [`UIGameDM.cpp`](UIGameDM.cpp.md).

**Needs** — [`UIGameMP.h`](UIGameMP.h.md)
**Used by** — [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) · [`UIGameDM.cpp`](UIGameDM.cpp.md) · [`UIGameTDM.cpp`](UIGameTDM.cpp.md) · [`UIGameTDM.h`](UIGameTDM.h.md) · [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) · [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the first mode-specific multiplayer interface, and the one the team modes derive
from. Substance in [`UIGameDM.cpp`](UIGameDM.cpp.md).

Its shape is the pattern for every mode: **nine captions plus a handful of readouts**, all
written into by the game mode and laid out entirely from authored files.

Exported units:

- `CUIGameDM` — the interface.
- `Init(stage)` — the three-stage construction; the ordering within stage 0 is load-bearing
  and is described in the twin.
- The nine caption setters — `SetTimeMsgCaption`, `SetSpectrModeMsgCaption`,
  `SetSpectatorMsgCaption`, `SetPressJumpMsgCaption`, `SetPressBuyMsgCaption`,
  `SetRoundResultCaption`, `SetForceRespawnTimeCaption`, `SetDemoPlayCaption`,
  `SetWarmUpCaption`. Each takes a **string-table identifier**, never display text.
- `SetVoteMessage`, `SetVoteTimeResultMsg` — the vote banner, created and destroyed with
  each vote rather than hidden.
- `SetFraglimit`, `SetRank`, `ChangeTotalMoneyIndicator`, `DisplayMoneyChange`,
  `DisplayMoneyBonus` — the readouts.
- `ShowFragList`, `ShowPlayersList` — two names for showing the scoreboard; they no longer
  differ.
- `UpdateTeamPanels` — mark the scoreboard as needing a refresh, deferring the cost until it
  is next shown.
- `flShowFragList` — a declared flag that nothing sets or tests.

**Notes** — the caption widgets and the money, rank, frag-limit and scoreboard members are
declared *protected* rather than private specifically so the team modes and the capture mode
can re-read them from their own layout files. That is the extension mechanism: a subclass
re-initializes the parent's widgets rather than replacing them.
