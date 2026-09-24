# src/xrGame/game_sv_base.h

> Declares the server half of the game rules: the interface every mode must satisfy, and the shared implementation of respawn points, map rotation and the delayed-event queue. Implemented in [`game_sv_base.cpp`](game_sv_base.cpp.md).

**Needs** — [`game_sv_base.cpp`](game_sv_base.cpp.md) · [`game_base.h`](game_base.h.md) · [`game_sv_event_queue.h`](game_sv_event_queue.h.md) · [`game_sv_item_respawner.h`](game_sv_item_respawner.h.md) · [`game_sv_base_console_vars.h`](game_sv_base_console_vars.h.md) · [`xrServerEntities/alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrNetServer/NET_Server.h`](../xrNetServer/NET_Server.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`game_sv_base.cpp`](game_sv_base.cpp.md) · [`game_sv_base_script.cpp`](game_sv_base_script.cpp.md) · [`game_sv_item_respawner.cpp`](game_sv_item_respawner.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md) · [`game_sv_single.h`](game_sv_single.h.md) · [`screenshot_server.cpp`](screenshot_server.cpp.md) · [`xrServer.h`](xrServer.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the server-side rules. Substance is in [`game_sv_base.cpp`](game_sv_base.cpp.md),
but **the interface here is itself substantive**: `IServerGameState` is what the engine
demands of any set of rules, and it is the widest such contract in the chapter. Reading it is
how you learn what a game mode is allowed to decide.

## the interface

**Contract** — what a rules implementation must supply, grouped by concern.

```text
INTERFACE ServerGameState                    # extends the shared GameState

  # Membership
  OnPlayerConnect / OnPlayerDisconnect / OnPlayerReady
  OnPlayerEnteredGame / OnPlayerConnectFinished

  # Match structure
  OnRoundStart / OnRoundEnd / SetRoundResult(reason)
  OnSwitchPhase(old, new)
  SaveMapList / HasMapRotation / MapRotation_AddMap / MapRotation_ListMaps
  OnNextMap / OnPrevMap / SwitchToNextMap

  # Voting
  IsVotingEnabled() / IsVotingEnabled(subject) / IsVotingActive / SetVotingActive
  OnVoteStart(command, sender) / OnVoteStop

  # Lookup — the naming policy below is load-bearing
  get_eid(entity_id) -> PlayerState        # who owns this body
  get_id(client_id)  -> PlayerState
  get_client(entity_id), get_id_2_eid(client_id), get_entity_from_eid(entity_id)
  get_name_id(client_id), get_players_count, get_alive_count(team), get_children(client_id)

  # Object lifecycle — the veto points
  OnPreCreate(entity) -> bool              # may refuse the spawn outright
  OnCreate(entity_id) / OnPostCreate(entity_id)
  OnTouch(who, what, forced) -> bool       # may refuse a pickup
  OnDetach(who, what)
  OnActivate(who, what) -> bool            # may refuse a use
  OnDestroyObject(entity_id)
  on_death(victim, killer)

  # Damage
  OnHit(hitter, target, packet)
  OnPlayerHitPlayer(hitter, target, packet)
  isFriendlyFireEnabled / CanHaveFriendlyFire

  # Respawn points
  assign_RP(entity, player)
  IsPointFreezed(point) / SetPointFreezed(point)

  # Network
  net_Export_State(packet, to)                  # a full snapshot
  net_Export_Update(packet, to, about)          # one player's increment
  net_Export_GameTime(packet)
  signal_Syncronize()                           # force a snapshot next tick

  # Persistence and level flow
  change_level / save_game / load_game / reload_game / switch_distance
  level_name(options)
  custom_sls_default / sls_default              # the mode's default save-load state

  # Misc
  Create(options) / Update / OnRender / DumpOnlineStatistic
  OnPlayerFire(client, packet) / OnPlayer_Sell_Item(client, packet)
  teleport_object / add_restriction / remove_restriction / remove_all_restrictions
```

**Invariants**

- **The three veto points — pre-create, touch and activate — are where the rules actually
  bite.** Everything else reports; these three can refuse. A mode enforces "you may not pick
  up the enemy's artefact" by refusing a touch, not by undoing one.
- Two of them are **pure virtual with no default** in the shared implementation — touch and
  detach, plus the friendly-fire capability question. Every mode must answer them; there is
  no sensible default for who may pick up what.
- The naming policy is documented in the header and must be honoured because three different
  kinds of integer are passed around: an *ordinal* is a position in the client list, an
  *identifier* is a client, an *entity identifier* is a body. The lookup functions exist in
  every pairing because the calling code holds whichever it happens to have.

## other exported units

- `ERoundEnd_Result` — why a round ended: finished normally, the game was restarted (two
  flavours, one of which skips the interlude), a limit was reached (time, frags or
  objectives), or forced. The reason governs whether the map rotates; see the implementation.
- `game_sv_GameState` — the shared implementation, holding the server handle, the delayed
  event queue, the item respawner, the map rotation list, the round-end reason, and the
  respawn point table.
- The respawn point table is **four** arrays wide, one per team slot, even though no mode has
  four teams. The extra slots exist because respawn points are authored with a team number
  and the level editor's numbering is one-based (see
  [`game_sv_capture_the_artefact_process_event.cpp`](game_sv_capture_the_artefact_process_event.cpp.md)).
- The player cap is 32, declared here and asserted on both sides of the snapshot.
- `parse_level_name` / `parse_level_version` — statics that pull the level and its version out
  of the server option string, callable before any rules object exists.
