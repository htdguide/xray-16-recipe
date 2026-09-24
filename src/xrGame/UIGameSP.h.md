# src/xrGame/UIGameSP.h

> Declares the single-player game UI and the level-change prompt implemented in [`UIGameSP.cpp`](UIGameSP.cpp.md).

**Needs** — [`UIGameCustom.h`](UIGameCustom.h.md) · [`UITimeDilator.h`](UITimeDilator.h.md) · [`ui/UIDialogWnd.h`](ui/UIDialogWnd.h.md)
**Used by** — [`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`GametaskManager.cpp`](GametaskManager.cpp.md) · [`UIGameSP.cpp`](UIGameSP.cpp.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`game_cl_single.cpp`](game_cl_single.cpp.md) · [`level_changer.cpp`](level_changer.cpp.md) · [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md) · [`stalker_animation_head.cpp`](stalker_animation_head.cpp.md) · [`UIGameTutorialSimpleItem.cpp`](ui/UIGameTutorialSimpleItem.cpp.md) · [`UIMainIngameWnd.cpp`](ui/UIMainIngameWnd.cpp.md) · [`UITalkWnd.cpp`](ui/UITalkWnd.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the two types whose substance is in [`UIGameSP.cpp`](UIGameSP.cpp.md), plus the
accessors for the process-wide time dilator.

Exported units:

- `CUIGameSP` — the single-player game UI: dialog routing, the objective banner, the
  trade/upgrade/talk/corpse-search entry points, and level change.
- `CChangeLevelWnd` — the confirmation prompt for crossing a level boundary; it pauses
  the simulation while visible and carries the destination it will send on confirmation.
- `TimeDilator()` / `CloseTimeDilator()` — reach and release the lazily created time
  dilator; see [`UITimeDilator.cpp`](UITimeDilator.cpp.md).
