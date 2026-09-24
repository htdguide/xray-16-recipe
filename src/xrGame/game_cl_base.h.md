# src/xrGame/game_cl_base.h

> Declares the client half of the game rules — the player table every client mirrors, and the wide set of hooks a mode overrides — implemented in [`game_cl_base.cpp`](game_cl_base.cpp.md).

**Needs** — [`game_cl_base.cpp`](game_cl_base.cpp.md) · [`game_base.h`](game_base.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`xrCore/client_id.h`](../xrCore/client_id.h.md)
**Used by** — [`DemoPLay_Control.cpp`](DemoPLay_Control.cpp.md) · [`EntityCondition.cpp`](EntityCondition.cpp.md) · [`GraviArtifact.cpp`](GraviArtifact.cpp.md) · [`HUDManager.cpp`](HUDManager.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md) · [`Level_input.cpp`](Level_input.cpp.md) · [`Level_load.cpp`](Level_load.cpp.md) · [`Level_network.cpp`](Level_network.cpp.md) · [`Level_network_messages.cpp`](Level_network_messages.cpp.md) · [`Level_network_spawn.cpp`](Level_network_spawn.cpp.md) · [`Level_network_start_client.cpp`](Level_network_start_client.cpp.md) · [`Level_start.cpp`](Level_start.cpp.md) · _and 36 more_
**Tier floor** — T3: a declaration

## Purpose

Declares the base every client-side game mode derives from. Substance is in
[`game_cl_base.cpp`](game_cl_base.cpp.md).

What the declaration itself decides is the **shape of a mode**: a long list of do-nothing
virtual hooks, each of which a mode fills or ignores. That list is the real interface
between the engine and a game mode, and a rebuild should read it as a specification of what
varying the rules is allowed to change. It divides into seven groups:

- **Identity and setup** — the mode's name, its initialisation, the team's configuration
  section, its own screen.
- **State intake** — apply a full state snapshot, apply an incremental update, apply a clock
  update, apply a signal.
- **Input** — keyboard press and release, returning whether the mode swallowed the key.
- **Chat and moderation** — say, and the four inbound message kinds (chat, warning, remote
  administration, and the vote pair).
- **Voting** — is voting enabled at all, is one running, start one, vote yes or no, and the
  two inbound notifications.
- **Rules predicates** — is this player my enemy, are these two entities enemies, may this
  actor sprint, is this player on that team, does the server arbitrate hits.
- **World events** — an object spawned, an object destroyed, a player's flags changed, a new
  player connected, a player voted, and the rendering hook.

Note that the enemy test is asked **twice, differently**: once of two player records and
once of two live entities. Deathmatch answers "everyone" to the first and consults the mode
for the second; the team modes answer both from the team field. A rebuild needs both,
because the two are asked from different places — the scoreboard asks about records, the
damage path asks about entities.

Exported units:

- `SZoneMapEntityData` — one marker on the minimap: a world position and a packed colour.
  Modes fill a list of these; the map screen draws them without knowing what they mean.
- `game_cl_GameState` — the base. It is also a *scheduled* object, so it gets a periodic
  update at a rate between 5 and 20 milliseconds rather than a per-frame one.
- The player table, keyed by client identifier, plus the local client's identifier and its
  own record held separately from the table.
- The weapon-usage statistics collector, owned here and shared by every mode.
- `lookat_player` — the record of whoever the camera is currently attached to, which is not
  the local player while spectating.
- `GetPlayerByGameID` / `GetPlayerByOrderID` / `GetClientIDByOrderID` / `GetPlayersCount` —
  the three ways the rest of the code addresses a player: by the entity identifier of his
  body, by position in the table, and by client identifier.
- `u_EventGen` / `u_EventSend` / `sv_GameEventGen` / `sv_EventSend` — building and sending
  an event upstream.
- `StartStopMenu` — a static that forwards to the current screen; a mode opens its menus
  through it.
