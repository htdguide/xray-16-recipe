# src/xrGame/Level.h

> Declares the game layer's level — the object that exists for as long as a match does — implemented across [`Level.cpp`](Level.cpp.md) and a dozen siblings.

**Needs** — [`xrEngine/IGame_Level.h`](../xrEngine/IGame_Level.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`xrEngine/StatGraph.h`](../xrEngine/StatGraph.h.md) · [`xrNetServer/NET_Client.h`](../xrNetServer/NET_Client.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`GlobalFeelTouch.hpp`](GlobalFeelTouch.hpp.md) · [`Level_network_map_sync.h`](Level_network_map_sync.h.md) · [`Level_network_Demo.h`](Level_network_Demo.h.md) · [`secure_messaging.h`](secure_messaging.h.md) · [`traffic_optimization.h`](traffic_optimization.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`Actor_Events.cpp`](Actor_Events.cpp.md) · [`Actor_Feel.cpp`](Actor_Feel.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`AnselManager.cpp`](AnselManager.cpp.md) · [`Artefact.cpp`](Artefact.cpp.md) · [`Car.cpp`](Car.cpp.md) · [`CarCameras.cpp`](CarCameras.cpp.md) · [`CarInput.cpp`](CarInput.cpp.md) · _and 222 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CLevel`. Substance is in [`Level.cpp`](Level.cpp.md) (the frame loop, the event
drain, correction prediction, the clock) and in its siblings for network start-up, save and
load, demo recording and input.

Two things this header decides are worth stating before any implementation is read.

**It is simultaneously the level and the network client.** The game's level object *is* the
connection endpoint: the message callbacks, the connection-failure callbacks and the send
path are all members. There is no separate client object to route through. A single-player
process still goes through all of it, connected to its own in-process server. A rebuild that
wants a level without a network stack must break that union first, and it runs deep.

**It is globally reachable, and the game layer reaches it constantly.** A free accessor
returns the one level, another returns the game rules through it. Hundreds of call sites in
the game layer use them instead of being handed a level. That is the single largest
obstacle to testing any of this in isolation.

Exported units — grouped by what they belong to rather than by order:

- `CLevel` — the level.
- `Level()` / `Game()` / `GameID()` — the global accessors, and the reason most of the game
  layer has no level parameter.
- `net_Start` / `net_Start_client` / `net_Stop` / `net_Load` / `net_Save` / `net_Update` —
  bringing a level up and down. The multi-stage start is declared here as numbered steps
  (six server-side, six client-side) so it can be driven one stage per frame with a loading
  screen between — a level takes too long to load in one frame.
- `Load_GameSpecific_Before` / `Load_GameSpecific_After` / `Load_GameSpecific_CFORM` and its
  serialize, deserialize and material-binding partners — the game layer's hooks into the
  engine's level load, chiefly to attach game material identities to collision triangles and
  to cache that binding.
- `OnFrame` / `OnRender` / `OnEvent` / `DumpStatistics` — the frame.
- `OnMessage` / `OnInvalidHost` / `OnInvalidPassword` / `OnSessionFull` / `OnConnectRejected`
  / `OnSessionTerminate` / `Send` — the transport surface.
- `ProcessGameEvents` / `ProcessGameSpawns` / `cl_Process_Event` / `cl_Process_Spawn` /
  `ProcessCompressedUpdate` / `game_events` / `game_spawn_queue` — the deferred event and
  spawn queues. Spawns are queued separately from events because a spawn may not happen
  inside another object's update.
- `GetInterpolationSteps` / `SetInterpolationSteps` / `InterpolationDisabled` /
  `ReculcInterpolationSteps` / `GetNumCrSteps` / `SetNumCrSteps` / `In_NetCorrectionPrediction`
  / `AddObject_To_Objects4CrPr` / `AddActor_To_Actors4CrPr` / `RemoveObject_From_4CrPr` /
  `PhisStepsCallback` — correction prediction and interpolation.
- `CurrentControlEntity` / `SetControlEntity` — which entity the player currently drives.
  Distinct from the entity the player *sees through*, which the engine's level owns.
- `GetStartGameTime` / `GetGameTime` / `GetEnvironmentGameTime` / `GetGameDateTime` /
  `GetDayTime` / `GetGameDayTimeMS` / `GetGameDayTimeSec` / `GetEnvironmentGameDayTimeSec` /
  `GetGameTimeFactor` / `SetGameTimeFactor` / `GetEnvironmentTimeFactor` /
  `SetEnvironmentTimeFactor` / `SetEnvironmentGameTimeFactor` — the two clocks, the
  simulation's and the weather's, each with its own rate.
- `IsServer` / `IsClient` / `Server` / `game` — authority. Not a partition: single player is
  both, demo playback inverts both.
- `space_restriction_manager` / `seniority_holder` / `client_spawn_manager` /
  `autosave_manager` / `ph_commander` / `ph_commander_scripts` / `MapManager` /
  `GameTaskManager` / `BulletManager` / `debug_renderer` — the per-level managers, each
  reached through an accessor that asserts existence rather than returning nothing. On a
  dedicated server most are absent and every use is guarded.
- `SLS_Load` / `SLS_Default` / `ClientSave` / `Objects_net_Save` — saved-game entry points.
- `g_cl_Spawn` / `g_sv_Spawn` / `spawn_item` — the three spawn paths: ask the server, obey
  the server, and create from a configuration section.
- `IR_On*` — the whole input surface: keyboard, text, mouse, controller, focus. The level is
  an input receiver, which is how gameplay input is routed while no menu is up.
- `PrefetchSound` / `static_Sounds` / `m_StaticParticles` — the per-level presentation
  resources the level owns outright.
- `OnAlifeSimulatorLoaded` / `OnAlifeSimulatorUnLoaded` — reset the identity-keyed stores
  when the simulation changes underneath.
- `name` / `version` / `GetLevelInfo` / `IsChecksumsEqual` / `CalculateLevelCrc32` — the
  level's identity and the checksum that refuses a mismatched client.
- `init_compression` / `deinit_compression` / `m_trained_stream` / `m_lzo_dictionary` /
  `m_lzo_working_buffer` — the network compression context. A *trained* statistical model
  and a shared dictionary, both built per level, because update traffic is repetitive in a
  way general-purpose compression cannot exploit.
- `OnSecureMessage` / `OnSecureKeySync` / `SecureSend` / `m_secret_key` — an encrypted
  channel over the same transport.
- `OnGameSpyChallenge` / `OnBuildVersionChallenge` / `OnConnectResult` /
  `SendClientDigestToServer` / `get_cdkey_digest` — the matchmaking-service handshake.
- `create_hud_zones_list` / `hud_zones_list` / `m_feel_deny` — the sensing helpers the level
  owns for want of a better home.
- `script_gc` / `remove_objects` / `ClearAllObjects` / `MakeReconnect` / `GetRealPing` /
  `get_RPID`.
- `AIStatistics` / `AIStats` — the AI budget counters: thinking, range queries, paths, node
  queries, vision split into portal traversal and ray casting, and script collection. The
  split is a statement of where the authors expected the AI's time to go.
- `OpenDemoFile` / `net_StartPlayDemo` and the demo surface textually included from
  [`Level_network_Demo.h`](Level_network_Demo.h.md).

## Notes

The demo-playback surface is brought in by textual inclusion in the middle of the class
rather than by composition or inheritance. It is a way of splitting one class across two
files; a rebuild should make it a member object.

`maxRP` (64) and `maxTeams` (32) are declared here and used by the commented-out respawn
lookup. Whether the limits still bind anywhere is not recoverable.
