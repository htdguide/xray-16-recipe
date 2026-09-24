# src/xrGame/UIGameAHunt.h

> Declares the artefact-hunt interface implemented in [`UIGameAHunt.cpp`](UIGameAHunt.cpp.md).

**Needs** — [`UIGameCustom.h`](UIGameCustom.h.md) · [`UIGameTDM.h`](UIGameTDM.h.md) · [`ui/UIDialogWnd.h`](ui/UIDialogWnd.h.md) · [`ui/UISpawnWnd.h`](ui/UISpawnWnd.h.md)
**Used by** — [`UIGameAHunt.cpp`](UIGameAHunt.cpp.md) · [`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md) · [`game_cl_artefacthunt.h`](game_cl_artefacthunt.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the artefact-hunt mode's interface: the team deathmatch's screen with two
additions. Substance in [`UIGameAHunt.cpp`](UIGameAHunt.cpp.md).

Exported units:

- `CUIGameAHunt` — the interface.
- `Init(stage)` — re-reads the parent's widgets from this mode's own layout, falling back to
  the deathmatch layout for the four widgets one shipped game omits.
- `SetReinforcementTimes` — the wave-respawn countdown; rendered as a shape or as a number
  depending on what the layout declared.
- `m_pBuySpawnMsgBox` — the prompt offering to pay for an early respawn, public because the
  game mode shows and hides it directly. Rebuilt on every session change, because its
  confirmation callback captures the session's mode.
- `SetBuyMsgCaption` — writes the string-table identifier for the "press buy" prompt.
- `SetClGame`, `UnLoad` — session binding and teardown.
