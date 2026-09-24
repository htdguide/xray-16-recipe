# src/xrGame/game_cl_teamdeathmatch.h

> Declares the team deathmatch client rules, implemented in [`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md).

**Needs** — [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md) · [`UIGameTDM.h`](UIGameTDM.h.md)
**Used by** — [`UIGameTDM.cpp`](UIGameTDM.cpp.md) · [`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md) · [`game_cl_artefacthunt.h`](game_cl_artefacthunt.h.md) · [`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md) · [`UISpawnWnd.cpp`](ui/UISpawnWnd.cpp.md) · [`UIVote.cpp`](ui/UIVote.cpp.md) · [`UIVotingCategory.cpp`](ui/UIVotingCategory.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares team deathmatch on the client, as a specialisation of deathmatch. Substance is in
[`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md).

Its own load-bearing content is the **team-number convention**, fixed here by the conversion
method: a player's team is one-based (1 or 2, with -1 meaning none), and every indexed
lookup into the two-team arrays goes through a conversion that subtracts one and passes -1
through. A rebuild that picks one numbering throughout deletes the whole method; a rebuild
that keeps two must keep the bridge explicit, because the two appear side by side in nearly
every method here.

Exported units:

- `game_cl_TeamDeathmatch` — the mode.
- `Init` / `OnConnected` / `createGameUI` / `SetGameUI` — bring-up: load both teams' data,
  instantiate the mode's screen, bind it.
- `net_import_state` — decode the snapshot, including the two friendly-identification
  permissions, and announce a tie broken or restored.
- `TranslateGameMessage` — the join-team and change-team feed messages.
- `shedule_Update` — per-update screen state, prompts and score.
- `OnRender` — teammate names and indicators.
- `UpdateMapLocations` / `GetMapEntities` — teammates on the map and the minimap.
- `IsEnemy` — against a player record, or between two live entities.
- `IsPlayerInTeam` / `ModifyTeam` / `GetTeamCount` — membership, the numbering bridge, and
  the constant two.
- `OnTeamSelect` / `OnGameMenuRespond_ChangeTeam` / `OnTeamChanged` — the team change
  request, reply and hook.
- `CanCallBuyMenu` / `CanCallSkinMenu` / `CanCallInventoryMenu` / `CanCallTeamSelectMenu` —
  the mutually exclusive screen gates, all of them behind team selection.
- `CanBeReady` — refuses, and opens team select, until a team is chosen.
- `OnMapInfoAccept` / `OnSkinMenuBack` / `OnTeamMenuBack` / `OnTeamMenu_Cancel` /
  `OnSpectatorSelect` — the pre-round navigation, including the modal trap on team select.
- `SetCurrentBuyMenu` / `SetCurrentSkinMenu` / `GetTeamMenu` / `getTeamSection` /
  `GetBaseCostSect` — the per-team menus and the configuration sections behind them.
- `OnSwitchPhase` / `OnSwitchPhase_InProgress` — win announcements, and the round-boundary
  rule that clears team selection only when the skin selection was also lost.
- `SetScore` / `GetGameScore` / `GetGreenTeamScore` / `GetBlueTeamScore` — the two-team score.
- `LoadSndMessages` / `PlayRankChangesSndMessage` — the fourteen announcements.
- `Set_ShowPlayerNames` / `Get_ShowPlayerNames` / `Get_ShowPlayerNamesEnabled` — the local
  preference and the server's permission, kept separate.
- `OnKeyboardPress` — the team key opens team select.
