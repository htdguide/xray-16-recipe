# src/xrGame/game_cl_capture_the_artefact.h

> Declares the capture-the-artefact client rules, implemented in [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) and two sibling files.

**Needs** — [`game_cl_mp.h`](game_cl_mp.h.md) · [`UIGameCTA.h`](UIGameCTA.h.md) · [`game_cl_capture_the_artefact_captions_manager.h`](game_cl_capture_the_artefact_captions_manager.h.md)
**Used by** — [`UIGameCTA.cpp`](UIGameCTA.cpp.md) · [`UIPlayerItem.cpp`](UIPlayerItem.cpp.md) · [`cta_game_artefact.cpp`](cta_game_artefact.cpp.md) · [`cta_game_artefact.h`](cta_game_artefact.h.md) · [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_capture_the_artefact_captions_manager.cpp`](game_cl_capture_the_artefact_captions_manager.cpp.md) · [`game_cl_capture_the_artefact_captions_manager.h`](game_cl_capture_the_artefact_captions_manager.h.md) · [`game_cl_capture_the_artefact_messages_menu.cpp`](game_cl_capture_the_artefact_messages_menu.cpp.md) · [`game_cl_capturetheartefact_buywnd.cpp`](game_cl_capturetheartefact_buywnd.cpp.md) · [`UIMainIngameWnd.cpp`](ui/UIMainIngameWnd.cpp.md) · [`UIMpTradeWnd_trade.cpp`](ui/UIMpTradeWnd_trade.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares capture the artefact on the client. Substance is in
[`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md), with the buy cycle
in [`game_cl_capturetheartefact_buywnd.cpp`](game_cl_capturetheartefact_buywnd.cpp.md) and
the radio phrases in
[`game_cl_capture_the_artefact_messages_menu.cpp`](game_cl_capture_the_artefact_messages_menu.cpp.md).

Two things the declaration itself fixes are worth naming before reading the implementation.

**It derives from the multiplayer base, not from team deathmatch.** Every team-mode feature
— team selection, skins, friendly identification, the two-team score — is re-implemented
here rather than inherited, and the conventions differ from the sibling mode's in small,
load-bearing ways. In particular its team-number conversion is the **identity**, where team
deathmatch's subtracts one, and the header carries a comment acknowledging that the team
indices are a known problem.

**Every objective accessor is guarded.** The artefact identifiers, owner identifiers, home
points and base radius are queried by the artefact entities themselves, which can exist
before the first snapshot arrives; each accessor asserts that a snapshot has been received.

Exported units:

- `game_cl_CaptureTheArtefact` — the mode.
- `Init` / `createGameUI` / `SetGameUI` / `OnConnected` — bring-up and screen binding.
- `net_import_state` / `net_import_update` — the snapshot and the frequent countdown update.
- `TranslateGameMessage` — the three objective events: taken, dropped, delivered.
- `PlayCapturedTheArtefact` / `PlayReturnedTheArtefact` / `PlayDeliveredTheArtefact` /
  `PlayRankChangedSnd` / `LoadSndMessages` — the announcement set, selected by the listener's
  team crossed with the actor's relationship to the listener.
- `shedule_Update` — prompts, money, countdowns and the end-of-round winner.
- `UpdateMoneyIndicator` — the watched player's balance, plus the local provisional spend.
- `UpdateMapLocations` — teammates and the four artefact marker kinds, rebuilt each pass.
- `OnSpawn` / `OnPlayerFlagsChanged` / `OnNewPlayerConnected` / `SetInvinciblePlayer` — the
  reactions to world and player-state changes, including base entry and exit edge detection.
- `OnRender` / `IsEnemy` / `Set_ShowPlayerNames` / `Get_ShowPlayerNames` /
  `Get_ShowPlayerNamesEnabled` — friendly identification, with the local preference and the
  server's permission kept separate.
- `PlayerCanSprint` — the artefact carrier's sprint restriction. The implementation tests the
  server's flag inverted; see the implementation twin.
- `CanActivateArtefact` — server policy read by the objective artefact.
- `OnKeyboardPress` / `OnKeyboardRelease` — the mode's keys; scores is hold-to-show.
- `CanCallBuyMenu` / `CanCallSkinMenu` / `CanCallTeamSelectMenu` / `CanCallInventoryMenu` /
  `CanBeReady` — the gates. Buying is allowed while dead, or while alive on a base.
- `NeedToSendReady_Actor` / `NeedToSendReady_Spectator` / `OnBuySpawnMenu_Ok` / `SpawnMe` —
  the reinforcement wave and the paid skip.
- `OnBuyMenu_Ok` / `OnBuyMenu_Cancel` / `OnBuyMenuOpen` / `LocalPlayerCanBuyItem` — the buy
  cycle, including the two events that tell the server the menu is open.
- `OnTeamSelect` / `OnGameMenuRespond_ChangeTeam` / `OnSkinMenu_Ok` /
  `OnGameMenuRespond_ChangeSkin` / `OnTeamChanged` / `OnRankChanged` / `OnTeamScoresChanged`
  — the two pre-round choices and the three things that invalidate the menus.
- `OnSpectatorSelect` / `OnMapInfoAccept` / `OnTeamMenuBack` / `OnTeamMenu_Cancel` /
  `OnSkinMenuBack` / `OnSkinMenu_Cancel` — the pre-round navigation.
- `OnVoteStart` / `OnVoteStop` / `OnVoteEnd` / `UpdateVotingTime` — voting, with the command
  string re-parsed and localised on the client.
- `UpdateWarmupTime` / `UpdateTimeLimit` / `InWarmUp` / `HasTimeLimit` /
  `Is_Rewarding_Allowed` — warm-up and the time limit. Nothing is rewarded during warm-up.
- `OnSwitchPhase` / `OnGameRoundStarted` — phase transitions and the round-start menu rebuild.
- `GetGreenArtefactID` / `GetBlueArtefactID` / `GetGreenArtefactOwnerID` /
  `GetBlueArtefactOwnerID` / `GetGreenArtefactRPoint` / `GetBlueArtefactRPoint` /
  `GetBaseRadius` — the guarded objective accessors the artefact entities consult.
- `GetGameScore` / `GetGreenTeamScore` / `GetBlueTeamScore` — the score.
- `GetLocalPlayerTeamSection` / `GetLocalPlayerTeam` / `getTeamSection` / `ModifyTeam` /
  `GetTeamCount` — the team lookups and the identity conversion.
- `OnSpeechMessage` — the authored radio phrases.
