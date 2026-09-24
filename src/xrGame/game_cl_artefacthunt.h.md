# src/xrGame/game_cl_artefacthunt.h

> Declares the artefact hunt client rules, implemented in [`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md).

**Needs** — [`game_cl_teamdeathmatch.h`](game_cl_teamdeathmatch.h.md) · [`UIGameAHunt.h`](UIGameAHunt.h.md)
**Used by** — [`UIGameAHunt.cpp`](UIGameAHunt.cpp.md) · [`UIPlayerItem.cpp`](UIPlayerItem.cpp.md) · [`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md) · [`UIStatsPlayerInfo.cpp`](ui/UIStatsPlayerInfo.cpp.md) · [`UIStatsPlayerList.cpp`](ui/UIStatsPlayerList.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares artefact hunt on the client, as a specialisation of team deathmatch. Substance is in
[`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md).

Its own load-bearing content is the **snapshot field set**, declared as plain public members
because the snapshot decoder writes them and the screen reads them directly: the artefact,
its bearer, the possessing team, the target count, and the reinforcement pair — each with a
previous-snapshot copy kept for one purpose only, cleaning up the map marker of an artefact
that has ceased to exist.

Exported units:

- `game_cl_ArtefactHunt` — the mode.
- `Init` / `createGameUI` / `SetGameUI` / `OnConnected` — bring-up and screen binding.
- `net_import_state` — decode the objective state; the reinforcement deadline is conditional
  on the interval being positive.
- `TranslateGameMessage` — the five objective events, each with a per-team, per-perspective
  announcement.
- `shedule_Update` — prompts, the reinforcement countdown, and closing the paid-spawn box.
- `PlayerCanSprint` — the bearer may not sprint, when the server says so.
- `UpdateMapLocations` / `GetMapEntities` — the artefact's single marker in one of three
  kinds, and its minimap point.
- `NeedToSendReady_Spectator` / `OnBuySpawnMenu_Ok` — the reinforcement wave and the paid
  skip.
- `CanCallBuyMenu` / `CanBeReady` — artefact hunt's own readiness rule, which deliberately
  does not extend team deathmatch's.
- `OnSellItemsFromRuck` — sell the whole backpack in one event, on base only.
- `SetScore` — team scores plus rank and the artefact target.
- `OnSpawn` / `OnDestroy` — the appearance particle effect.
- `SendPickUpEvent` — bypasses both intermediate modes, because taking the artefact is the
  objective and must not be restricted.
- `getTeamSection` / `GetBaseCostSect` — the mode's configuration sections.
- `LoadSndMessages` — the fourteen announcements.
