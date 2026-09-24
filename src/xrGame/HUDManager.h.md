# src/xrGame/HUDManager.h

> Declares the game's heads-up-display object, implemented in [`HUDManager.cpp`](HUDManager.cpp.md).

**Needs** — [`xrEngine/CustomHUD.h`](../xrEngine/CustomHUD.h.md) · [`HitMarker.h`](HitMarker.h.md) · [`Level.h`](Level.h.md)
**Used by** — [`Actor_Feel.cpp`](Actor_Feel.cpp.md) · [`CustomDetector.cpp`](CustomDetector.cpp.md) · [`GamePersistent.cpp`](GamePersistent.cpp.md) · [`HUDManager.cpp`](HUDManager.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`Level_network_start_client.cpp`](Level_network_start_client.cpp.md) · [`UIFrameRect.cpp`](UIFrameRect.cpp.md) · [`level_script.cpp`](level_script.cpp.md) · [`player_hud_tune.cpp`](player_hud_tune.cpp.md) · [`UIFrameLine.cpp`](ui/UIFrameLine.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CHUDManager`, the game module's filling of the engine's display hook, and the
global accessor the rest of the chapter reaches it through — it hangs off the level, so
there is one per session. Substance is in [`HUDManager.cpp`](HUDManager.cpp.md).

Exported units:

- `CHUDManager` — the object.
- `Render_First`, `Render_Last` — the two first-person-weapon passes, before and after
  the world.
- `RenderUI` — the two-dimensional layer: markers, screens, text, cursor, pause banner.
- `RenderActiveItemUIQuery`, `RenderActiveItemUI` — the opt-in pass for interface
  attached to the held item.
- `OnFrame` — tick the screens and re-cast the look-at ray.
- `Load`, `OnConnected`, `OnDisconnected`, `OnUIReset` — build the screen set, the
  online gate, and the in-place rebuild after a resolution or language change.
- `GetGameUI` — the screen set.
- `GetCurrentRayQuery` — what the player is looking at; read by the depth-of-field
  effector among others.
- `SetCrosshairDisp`, `ShowCrosshair`, `SetFirstBulletCrosshairDisp` — the reticle's
  inputs, including the two-dispersion form that honours the "static reticle" setting.
- `HitMarked`, `AddGrenade_ForMark`, `Update_GrenadeView`, `SetHitmarkType`,
  `SetGrenadeMarkType` — the damage-direction and grenade-warning markers.
- `SetRenderable` — the global "draw the HUD at all" flag.
- `net_Relcase` — forward an impending destruction to everything that may hold the
  object.
- `HUD()` — the global accessor, reached through the level.

## Notes

The declaration carries a mutex guarding the two render brackets, with the original's own
note that it should not be needed. See [`HUDManager.cpp`](HUDManager.cpp.md) for what it
protects and why a rebuild does not need it.
