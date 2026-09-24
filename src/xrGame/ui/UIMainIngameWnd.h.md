# src/xrGame/ui/UIMainIngameWnd.h

> Declares the heads-up overlay: the minimap, the condition warning icons, the quick-use
> slots, the pick-up preview and the look-at hint that sit over the running game.

**Needs** — [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md) · [`UIGameLog.h`](UIGameLog.h.md) · [`UIMotionIcon.h`](UIMotionIcon.h.md) · [`UIZoneMap.h`](../UIZoneMap.h.md) · [`UIHudStatesWnd.h`](UIHudStatesWnd.h.md) · [`UIArtefactPanel.h`](UIArtefactPanel.h.md) · [`HudSound.h`](../HudSound.h.md) · [`EntityCondition.h`](../EntityCondition.h.md) · [`xrServerEntities/alife_space.h`](../../xrServerEntities/alife_space.h.md)
**Used by** — [`ActorCondition.cpp`](../ActorCondition.cpp.md) · [`Actor_Feel.cpp`](../Actor_Feel.cpp.md) · [`UIGameCustom.cpp`](../UIGameCustom.cpp.md) · [`actor_communication.cpp`](../actor_communication.cpp.md) · [`game_cl_artefacthunt.cpp`](../game_cl_artefacthunt.cpp.md) · [`game_cl_capture_the_artefact.cpp`](../game_cl_capture_the_artefact.cpp.md) · [`game_cl_teamdeathmatch.cpp`](../game_cl_teamdeathmatch.cpp.md) · [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) · [`UIActorMenu_script.cpp`](UIActorMenu_script.cpp.md) · [`UIActorStateInfo.cpp`](UIActorStateInfo.cpp.md) · [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md) · [`UIMotionIcon.cpp`](UIMotionIcon.cpp.md) · [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md), plus two
small enumerations that are part of the contract rather than of the implementation.

## Exported units

- **The overlay window** — built once per session, drawn and updated every frame while the
  player is in the world.
- `EWarningIcons` — the set of icons that live in the warning strip: a wildcard meaning *all
  of them*, the jammed-weapon icon, the invincibility icon and the artefact-carrier icon. The
  wildcard is the first value and the rest follow in order, which the threshold loader relies
  on; see the twin.
- `EFlashingIcons` — the two attention icons: a new task, and new mail.
- `Init`, `Draw`, `Update` — the lifecycle the game layer drives.
- `ShowZoneMap` / `IsZoneMapShown` / `DrawZoneMap` / `UpdateZoneMap` — minimap visibility and
  out-of-band drawing, used by screens that want the minimap on top of themselves.
- `SetWarningIconColor` / `TurnOffWarningIcon` — the only way a warning icon changes state.
- `SetFlashIconState_` — turn one attention icon on or off.
- `UpdateMainIndicators` / `UpdateBoosterIndicators` / `DrawMainIndicatorsForInventory` — the
  condition strip, recomputed here and re-drawn by the inventory screen.
- `SetPickUpItem` — nominate the item the crosshair is over for the pick-up preview.
- `ReceiveNews` — hand a news record to the message window and wake the PDA.
- `AnimateContacts` — restart the contact-counter flash, optionally with a sound.
- `SetMPChatLog` — adopt the multiplayer chat and log windows for update purposes only.
- `OnConnected`, `OnSectorChanged`, `reset_ui` — the three world-level events the overlay
  cares about.
- `m_Thresholds` — the per-icon colour-change thresholds read from configuration.
