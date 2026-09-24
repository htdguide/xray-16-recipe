# src/xrGame/game_sv_capture_the_artefact.h

> Declares capture the artefact: two teams, two artefacts, two bases — a sibling of team deathmatch rather than a descendant, which is why it declares its own teams, its own balancing, its own warm-up and its own buy cycle.

**Needs** — [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) · [`game_sv_capture_the_artefact_buy_event.cpp`](game_sv_capture_the_artefact_buy_event.cpp.md) · [`game_sv_capture_the_artefact_myteam_impl.cpp`](game_sv_capture_the_artefact_myteam_impl.cpp.md) · [`game_sv_capture_the_artefact_process_event.cpp`](game_sv_capture_the_artefact_process_event.cpp.md) · [`game_sv_mp.h`](game_sv_mp.h.md) · [`actor_mp_server.h`](actor_mp_server.h.md) · [`xrServer.h`](xrServer.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`xrServerEntities/xrServer_Object_Base.h`](../xrServerEntities/xrServer_Object_Base.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`game_sv_capture_the_artefact_process_event.cpp`](game_sv_capture_the_artefact_process_event.cpp.md)
**Tier floor** — T2: a class declaration over session rules

## Purpose

The rules are in [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md)
and two companion files. What this declaration decides, and what nothing else states, is the
mode's **position in the hierarchy**: it derives straight from the multiplayer base, not from
team deathmatch, even though it is a two-team objective mode with friendly fire, team
balancing, automatic team swapping, warm-up, a buy economy and a spectator camera — every
one of which team deathmatch already has.

The consequence is that all of those are re-declared and re-implemented here, in parallel with
the versions in [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md) and
[`game_sv_teamdeathmatch.h`](game_sv_teamdeathmatch.h.md). They read the *same* console
variables, so an operator setting friendly fire or auto-balance affects both chains, but the
code implementing them is separate and the two have diverged. A rebuild should lift the
shared machinery out of the mode hierarchy and leave the modes holding only their rules; that
is a larger change than it looks, and it is the single biggest structural improvement
available in this chapter.

The mode is also the most *private* class in the group: almost everything is hidden, including
every rule a descendant of the other chain would override. Nothing derives from it, and
nothing is meant to.

## State

```text
RECORD CaptureTheArtefactServer EXTENDS MultiplayerServer
  teams              : map<TeamId, TeamState>   # EXACTLY two entries: green and blue
                                                # TeamState is in the myteam file
  teams_swapped      : bool                     # this map's sides have been mirrored once

  # --- anomalies: this mode's own rotation, unrelated to deathmatch's ---
  anomaly_permanent  : list<(text, int)>        # (zone name, chosen entity id)
  anomaly_sets       : list<(list<(text,int)>, bool)>   # a set and whether it is live
  anomaly_ids        : multimap<text, (int id, int use_count)>
                       # every spawned zone, by name; use_count spreads the picks
  last_anomaly_start : int (ms)

  # --- the asynchronous buy cycle ---
  buy_menu_states    : map<client, BuyMenuState>
  dead_buyers        : map<client, int>   # 1 once this dead player has bought since dying
  money_for_buy_spawn: int (signed)       # negative; default -10000
  not_free_ammo      : text               # sections excluded from the free magazine

  # --- timing ---
  round_started           : bool
  next_reinforcement_time : int (ms)      # TWO MEANINGS — see Invariants
  current_time            : int (ms)      # the server clock, sampled once per update
  invincibility_deadlines : map<client, int (ms)>   # 0 means already expired
  warmup_deadline         : int (ms) ; in_warmup : bool

  # --- the server's own spectator camera ---
  spectator_mode : bool ; spectator_target : GameObject
  spectator_switch_at : int (ms) ; spectator_switch_delta : int (ms)

ENUM BuyMenuState
  CLOSED           # set when the client reports the menu closed
  OPEN             # set when the client reports it opened; only a dead player may
  READY_TO_SPAWN   # set by a wave that arrived while the menu was open
```

Invariants:

- **The team map holds exactly two entries and is indexed by the team enumeration**, whose
  values are green = 0, blue = 1, spectators = 2. Spectators are a team value a player can
  hold but never a key in this map, so every lookup of a player's team must tolerate a miss.
- **`next_reinforcement_time` means two different things in two phases.** While a round runs
  it is the next respawn wave; in the scores phase it is the deadline for the end-of-round
  dwell. The field is reused because the two are never live at once, and a rebuild should
  give them separate names — the overloading is the kind of thing that survives a refactor
  and then produces a round that ends in three seconds.
- **`current_time` is a cached sample, not a clock.** It is written once per update in the
  in-progress and scores phases only, and read by the incremental export. During the pending
  phase it is never refreshed, so the reinforcement countdown exported to a joining client is
  computed against a stale sample. A rebuild should read the clock where it is used.
- **The invincibility deadline map is never pruned.** Entries are zeroed when they expire and
  removed only on disconnect, so the map grows to the number of distinct clients seen. Bounded
  in practice by the player cap; still a leak in shape.
- **The anomaly identifier map is a multimap with a use counter**, because a level names a
  zone *type* in a set and may contain many zones of that type. Choosing the least-used
  instance is what spreads a set across the map instead of stacking it on one zone.

## `game_sv_CaptureTheArtefact` — the rule surface

**Contract** — everything this mode decides, grouped by concern. Almost all of it is private:
the class is a leaf and its rules are not offered for override.

```text
# --- The objective ---
on_touch(who, target, forced) -> bool     # take, return, or refuse
on_detach(who, target)                    # drop
on_activate(who, target) -> bool          # activate, permitted only on your own artefact
check_for_artefact_delivering()           # a carrier standing on his own base point
check_for_artefact_returning(now)         # a loose artefact walks itself home
actor_deliver_artefact_on_base(actor, actor_team, artefact_team)
drop_artefact(owner, artefact, position)
respawn_artefacts() / move_artefact_to_point(artefact, point)
return_artefact_to_base()                 # DECLARED AND NEVER DEFINED — see Notes

# --- The round ---
check_for_round_start() / check_for_round_end() / check_for_all_players_ready()
start_new_round()                         # a DELIVERY's reset, not a round boundary
prepare_client_for_new_round(client) / move_life_actors() / respawn_dead_players()
respawn_client(client) / clear_ready_flag_from_all()
check_for_warmup(now)

# --- Membership ---
on_player_change_team(player, team) / on_player_change_skin(client, skin)
balance_teams() / swap_teams()
load_team_data(team, section) / load_artefact_rpoints()

# --- The asynchronous buy cycle ---
on_player_open_buy_menu(client) / on_player_close_buy_menu(client)
on_player_buy_finished(client, packet) / on_player_buy_spawn(client)
check_if_player_in_buy_menu(client) / set_ready_to_spawn_player(client)
on_close_buy_menu_from_all()

# --- Damage and scoring ---
get_kill_result / on_kill_result / on_give_bonus
on_player_hit_player / on_player_hit_player_case
reset_timeout_invincibility(now) / reset_invincibility(client)
player_check_rank(player)

# --- Anomalies (this mode's own copy) ---
load_anomaly_set() / restart_random_anomaly() / stop_previous_anomalies()
send_anomaly_states() / check_anomaly_update(now) / min_used_anomaly_id(name)
```

**Invariants** — **`on_activate` is a veto point this mode is the only user of.** The other
modes never override it. Here it answers "may this player trigger his own team's artefact",
and it is what makes an artefact a thing you do something to rather than only carry.

**Notes** — `return_artefact_to_base` is declared and has no definition anywhere. The
behaviour it names exists — a loose artefact walks itself home on a timer — but it is written
inline in the returning check. A declaration with no body is dead surface; a rebuild drops it.

## The match tunables

**Contract** — the settings, and where each comes from. Capture the artefact reads **three
other modes' console variables** as well as its own, because it needs their rules without
inheriting their code.

```text
# --- this mode's own ---
invincibility_time      = 5 seconds     # option "dmgblock"
artefact_returning_time = 45 seconds    # option "artrettime"
activated_artefact_ret  = 0             # option "actret"; non-zero changes what touching
                                        # your own displaced artefact does — see the .cpp
player_scores_delay     = 3 seconds     # the end-of-round dwell
artefact_base_radius    = 1.0           # how close counts as "at the base point"
rank_up_arts_divisor    = 1             # gates promotion on the leading score

# --- borrowed from deathmatch ---
anomalies_enabled ("ans") / anomaly_set_length ("anslen", MINUTES)
pda_hunt ("pdahunt") / damage_block_indicators ("dmbi")
warmup_time ("warmup") / time_limit (minutes)

# --- borrowed from team deathmatch ---
auto_balance ("abalance") / auto_swap ("aswap")
friendly_indicators ("fi") / friendly_names ("fn")
friendly_fire_modifier ("ffire") / team_kill_limit / team_kill_punishment

# --- borrowed from artefact hunt ---
score_limit ("anum")        # deliveries to win
reinforcement_time ("reinf", SECONDS)
bearer_cannot_sprint
```

**Invariants** — **the reinforcement interval is clamped to at least one second on read, and
the millisecond accessor substitutes one second for zero.** Artefact hunt's two sentinels —
zero for "no waves" and minus one for "waves on delivery" — are therefore *unreachable* here.
Capture the artefact always has waves, minimum one second apart. A rebuild must not copy
artefact hunt's three-valued setting into this mode; the two disjuncts that test for the
sentinels are dead code.

**The mode has no wave gate of its own.** A dead player who sends a ready is respawned
immediately, whatever the wave timer says. The gate that makes waves feel like waves lives
**on the client**, which intercepts the spawn key while dead and offers the paid-spawn
confirmation instead of readying (see
[`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md)). So the paid
spawn is a client-side courtesy, not a server rule: a client that simply sends a ready skips
the wave for nothing. Compare artefact hunt, which enforces the same rule on the server. A
rebuild must move the gate server-side — this is the clearest example in the chapter of a
rule placed on the untrusted end.

**Notes** — the friendly-fire question is answered by testing whether the modifier, scaled by a
hundred and truncated to an integer, is above zero. A modifier below one percent therefore
reads as friendly fire being off while still scaling damage. Harmless with the shipped values
and worth not reproducing.

## The buy-menu state machine

**Contract** — the three-state record above, per client, driving an **asynchronous** buy
cycle: a dead player may shop while the wave he is waiting for arrives, and his respawn is
deferred until he closes the menu.

```text
open buy menu   (dead player only)      -> OPEN
a wave arrives while OPEN               -> READY_TO_SPAWN   (he is NOT respawned yet)
client closes the menu, state OPEN      -> CLOSED
client closes the menu, READY_TO_SPAWN  -> respawn him now, then CLOSED
round start                             -> every entry discarded
```

**Invariants** — a player shopping when his wave arrives **misses no time**: the wave marks him
and he spawns the instant he closes the menu. Without the intermediate state he would either
be respawned with the menu open — losing the purchase — or skipped until the next wave.

The states are keyed on the client record rather than on the player, so a client that
disconnects with the menu open leaves a stale entry; the round start clears them all.

**Notes** — the separate `dead_buyers` flag answers a different question: *has this dead player
bought anything since he died?* If he has, his respawn must not overwrite his purchase with
the rank's default loadout. Two flags about the same menu, tracking two different facts, and
both are needed.

## `type_name`

**Contract** — `capturetheartefact`. Advertised to the server browser and compared against by
scripts, so frozen as text.
