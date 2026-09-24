# src/xrGame/GamePersistent.h

> Declares the game module's process-lifetime object, implemented in [`GamePersistent.cpp`](GamePersistent.cpp.md).

**Needs** — [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`player_hud_tune.h`](player_hud_tune.h.md)
**Used by** — [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`CustomZone.cpp`](CustomZone.cpp.md) · [`EffectorFall.cpp`](EffectorFall.cpp.md) · [`GamePersistent.cpp`](GamePersistent.cpp.md) · [`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md) · [`Level_load.cpp`](Level_load.cpp.md) · [`Missile.cpp`](Missile.cpp.md) · [`Weapon.cpp`](Weapon.cpp.md) · [`ZoneCampfire.cpp`](ZoneCampfire.cpp.md) · [`actor_memory.cpp`](actor_memory.cpp.md) · [`actor_mp_client_import.cpp`](actor_mp_client_import.cpp.md) · _and 8 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CGamePersistent`, the game module's filling of the engine's process-lifetime
interface, plus the global accessor every other file in the chapter reaches it through.
Substance is in [`GamePersistent.cpp`](GamePersistent.cpp.md).

Exported units:

- `CGamePersistent` — the object itself.
- `CreateLevel` / `DestroyLevel` — the game module's answer to "what is a level".
- `PreStart`, `Start`, `Disconnect` — session bracket.
- `OnAppStart`, `OnAppEnd`, `OnGameStart`, `OnGameEnd` — process and session lifecycle.
- `OnAppActivate`, `OnAppDeactivate` — window focus, and the pause policy around it.
- `OnFrame`, `OnEvent` — the per-frame hook and the deferred-event handler.
- `UpdateGameType` — re-derive the game mode and the active key-binding group.
- `CanBePaused` — the single authority on whether pausing is allowed.
- `OnRenderPPUI_query` / `_main` / `_PP` — the post-process UI pass, delegated to the
  main menu.
- `GetCurrentDof`, `SetBaseDof`, `SetEffectorDOF`, `RestoreEffectorDOF`,
  `SetPickableEffectorDOF` — the depth-of-field effector's push interface.
- `OnSectorChanged`, `OnAssetsChanged` — notifications from the renderer and the
  virtual filesystem.
- `DumpStatistics` — debug overlay contribution.
- `GetHudTuner` — hands out the first-person-model tuning helper, a development tool.
- `GamePersistent()` — the global accessor. This is the service-locator pattern the
  preface warns about: the object is reachable from anywhere in the chapter through one
  mutable global filled at startup.

## Notes

The declaration exposes the demo playlist's reader and its change timer as public fields
rather than behind methods. Nothing outside the file legitimately writes them; treat them
as internal in a rebuild.
