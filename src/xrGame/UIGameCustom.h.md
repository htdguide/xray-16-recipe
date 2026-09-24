# src/xrGame/UIGameCustom.h

> Declares the in-game interface base, the timed text overlay, and the multiplayer map catalogue — implemented in [`UIGameCustom.cpp`](UIGameCustom.cpp.md).

**Needs** — [`UIDialogHolder.h`](UIDialogHolder.h.md) · [`inventory_space.h`](../xrServerEntities/inventory_space.h.md) · [`gametype_chooser.h`](../xrServerEntities/gametype_chooser.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`Common/object_interfaces.h`](../Common/object_interfaces.h.md) · [`xrEngine/CustomHUD.h`](../xrEngine/CustomHUD.h.md) · [`xrCommon/xr_string.h`](../xrCommon/xr_string.h.md)
**Used by** — [`GamePersistent.cpp`](GamePersistent.cpp.md) · [`HUDManager.cpp`](HUDManager.cpp.md) · [`Inventory.cpp`](Inventory.cpp.md) · [`InventoryBox.cpp`](InventoryBox.cpp.md) · [`Level_input.cpp`](Level_input.cpp.md) · [`Level_start.cpp`](Level_start.cpp.md) · [`UIDialogHolder.cpp`](UIDialogHolder.cpp.md) · [`UIGameAHunt.h`](UIGameAHunt.h.md) · [`UIGameCustom.cpp`](UIGameCustom.cpp.md) · [`UIGameCustom_script.cpp`](UIGameCustom_script.cpp.md) · [`UIGameMP.cpp`](UIGameMP.cpp.md) · [`UIGameMP.h`](UIGameMP.h.md) · [`UIGameSP.cpp`](UIGameSP.cpp.md) · [`UIGameSP.h`](UIGameSP.h.md) · _and 22 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares the base every game mode's in-game screen set derives from, plus two things that
have no business here and are declared here anyway. Substance in
[`UIGameCustom.cpp`](UIGameCustom.cpp.md); the script surface in
[`UIGameCustom_script.cpp`](UIGameCustom_script.cpp.md).

Exported units:

- `CUIGameCustom` — the in-game interface. It is a **screen stack and input router**
  (inheriting the dialog holder), a **frame-signal subscriber**, and a **device-reset
  subscriber** — the third because every screen holds textures that must be rebuilt when
  the graphics device is lost. One instance per session; the game mode's subclass adds its
  own panels on top.
  - `Init(stage)` — **three-stage construction**, and the staging is the load-bearing part:
    stage 0 creates what every mode shares, stage 1 reads each mode's own layout, stage 2
    attaches the mode's panels to the window after the shared ones. A subclass that calls
    the base at the wrong stage gets its panels behind the shared ones.
  - `Render`, `OnFrame`, `OnUIReset` — draw, advance, rebuild after a device reset.
  - `GetActorMenu`, `GetPdaMenu`, `ShowActorMenu`, `HideActorMenu`, `UpdateActorMenu`,
    `ShowPdaMenu`, `HidePdaMenu`, `UpdatePda`, `CurrentItemAtCell` — the two large screens
    and the one query scripts use to ask what the cursor is over.
  - `AddCustomStatic`, `GetCustomStatic`, `RemoveCustomStatic`, `CommonMessageOut` — the
    script-driven text overlay: named, optionally single-instance, optionally self-expiring.
  - `ShowMessagesWindow`, `HideMessagesWindow`, `ShowGameIndicators`,
    `GameIndicatorsShown`, `ShowCrosshair`, `CrosshairShown` — visibility switches. The
    crosshair is stored in the engine's heads-up-display flags rather than here, because the
    renderer reads it.
  - `SetClGame`, `OnConnected`, `Load`, `UnLoad` — the session's own lifecycle.
  - `ChangeTotalMoneyIndicator`, `DisplayMoneyChange`, `DisplayMoneyBonus`,
    `HideShownDialogs`, `ReinitDialogs` — **empty in the base**; these are the hooks a
    multiplayer mode fills. Single player has no money indicator.
  - `update_fake_indicators`, `enable_fake_indicators` — a script-driven override of the
    condition readouts, so a scripted sequence can show the player a value their character
    does not actually have.
  - `OnInventoryAction` — the single hook by which inventory changes reach the interface.
  - `GetDebugType`, `FillDebugTree`, `FillDebugInfo` — the debug overlay's view of this
    window tree.

- `StaticDrawableWrapper` — one script-created text overlay: a text widget, an expiry time
  and a name. Its `IsActual` is the lifetime test; an overlay with no expiry lives until
  removed by name.

- `CMapListHelper`, `SGameTypeMaps`, `MPLevelDesc`, `MPWeatherDesc`, and the process-wide
  instance — **the multiplayer map and weather catalogue**, which has nothing to do with the
  in-game interface. It is built by scanning the installed level archives and is consulted
  by the lobby. It lives here because the lobby had no better owner; a rebuild moves it.

- `CurrentGameUI` — the process-wide accessor. The interface is reached globally rather than
  passed, which is the engine's habit throughout.
