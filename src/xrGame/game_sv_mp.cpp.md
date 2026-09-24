# src/xrGame/game_sv_mp.cpp

> Everything every multiplayer mode's authoritative side needs and none of the mode's own rules: rounds, respawns, corpses, ranks and money, voting, bans, map rotation, and the statistics dump.

**Needs** — [`game_sv_mp.h`](game_sv_mp.h.md) · [`game_sv_base.h`](game_sv_base.h.md) · [`game_sv_mp_team.h`](game_sv_mp_team.h.md) · [`game_sv_mp_vote_flags.h`](game_sv_mp_vote_flags.h.md) · [`game_base_kill_type.h`](game_base_kill_type.h.md) · [`game_base_menu_events.h`](game_base_menu_events.h.md) · [`actor_mp_server.h`](actor_mp_server.h.md) · [`cdkey_ban_list.h`](cdkey_ban_list.h.md) · [`player_name_modifyer.h`](player_name_modifyer.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrGameSpyServer.h`](xrGameSpyServer.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Spectator.h`](Spectator.h.md) · [`Grenade.h`](Grenade.h.md) · [`Artefact.h`](Artefact.h.md) · [`Inventory.h`](Inventory.h.md) · [`WeaponKnife.h`](WeaponKnife.h.md) · [`MPPlayersBag.h`](MPPlayersBag.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`date_time.h`](date_time.h.md) · [`ui/UIBuyWndShared.h`](ui/UIBuyWndShared.h.md) · [`xrServerEntities/xrServer_Object_Base.h`](../xrServerEntities/xrServer_Object_Base.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [`xrCore/xr_token.h`](../xrCore/xr_token.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)

**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: session and entity bookkeeping over the server; the frozen packet layouts are the only hard edge

## Purpose

Every multiplayer mode — deathmatch, team deathmatch, artefact hunt, capture the artefact —
derives from this class, and it is the layer at which "a match" exists as opposed to "a
game". It owns seven things that are the same whatever the mode is:

1. **The round boundary.** Starting a round destroys every entity and rebuilds the level from
   its authored spawn records; ending one freezes the world and stops accepting events.
2. **Respawn.** A killed player becomes a corpse, then a spectator, then an actor again, and
   each of those is a distinct **server object** substitution under the same client.
3. **Corpses.** Bodies accumulate and are reaped oldest-first once there are too many.
4. **Rank, experience and money.** A ladder loaded from configuration, experience earned by
   damage weighted by the victim's rank, and a money ledger with itemised bonuses.
5. **Voting.** A command string proposed by a player, tallied over a window, and executed
   through the console if it passes.
6. **Bans and kicks**, keyed by the player's account digest rather than by address.
7. **Statistics**, dumped to configuration-format files on a timer and at round end.

Mode-specific rules — what scores, when a round ends, who may buy — are left as empty or
trivial hooks for the derived modes to fill.

## State

```text
RECORD MultiplayerServer EXTENDS GameStateServer
  corpse_list       : queue<int (16-bit)>   # entity ids, oldest first
  ranks             : list<Rank>            # the ladder, loaded once
  rank_up_allowed   : bool                  # modes disable promotion during warm-up
  team_list         : list<TeamData>        # skins, default items, money floor, per team
  item_registry     : ItemRegistry          # section name <-> small index, for the buy protocol
  spectator_modes   : int (8-bit)           # which spectator cameras this server permits
  cdkey_ban_list    : BanList

  voting_active     : bool
  voting_real       : bool                  # a known command, as opposed to a raw console line
  vote_start_time   : int (ms)              # server time
  vote_command      : text                  # what is executed if it passes
  voting_string     : text                  # what the clients are shown
  started_player    : text

  async_stats       : map<client, bool>     # who has answered the statistics request
  stats_request_time: int (ms)
  round_dump_path   : text                  # empty when no round dump is open

RECORD Rank
  title              : text
  terms              : list<int>            # experience thresholds; at most 2 per rank
  bonus_money        : int                  # paid on promotion
  rank_diff_exp_bonus: list<real>           # multiplier per VICTIM rank
```

Invariants:

- **The corpse list holds entity identifiers, not entities.** A corpse may already be gone
  when its turn comes, so every reap re-resolves and drops stale entries.
- **A corpse with children is never reaped.** Destroying a body that still owns items would
  orphan them; the reap skips it and comes back next update.
- **`rank_diff_exp_bonus` is indexed by the *victim's* rank**, not the attacker's. It lives on
  the attacker's rank record, so the lookup is two-dimensional: attacker rank picks the
  table, victim rank picks the entry.
- **Money is clamped to the team's floor and to one million.** The floor is per team and can
  be negative, which is what lets a mode run players into debt.

## `Update`

**Contract** — the server's per-update pass: reap surplus corpses, advance any vote, push
money changes to the clients who have them, and dump statistics on the configured period.

```text
FUNCTION update()
  base.update()
  i = 0
  WHILE i < corpse_list.length
    IF corpse_list.length <= max_corpses: BREAK      # only the surplus is reaped
    id = corpse_list[i]
    corpse = server_object(id)
    IF corpse is none: remove entry ; CONTINUE       # already gone
    IF corpse still has children: i = i + 1 ; CONTINUE
    broadcast destroy(id) ; remove entry

  IF voting is enabled and active: update_vote()
  update_players_money()
  IF a statistics period is set AND that many minutes have passed
     AND the round is in progress
    dump the online statistics and the round statistics
```

**Notes** — the reap keeps the **newest** corpses and destroys the oldest, capped at ten by
default. That cap is a memory and rendering budget, not a rule; a rebuild may tune it, but
the shape — bodies persist until pressure forces a reap, rather than expiring on a timer —
is what makes a firefight's aftermath visible.

The "has children" skip is why a corpse holding a dropped weapon outlives the cap. It is
also why the loop advances the index in that case instead of removing: the entry must be
retried.

The statistics timer is in whole minutes of wall clock and is compared against a global
last-dump minute, so several modes running in one process would interfere. There is only
ever one.

## `OnRoundStart`

**Contract** — begins a round. Clears the weapon statistics and opens a new round dump,
empties the corpse list, switches to the in-progress phase, stamps the round number and
start time, clears every player's ready and dead flags, discards the delayed event queue,
**destroys every entity in the level and rebuilds it from the authored spawn records**, and
broadcasts a round-started message.

```text
FUNCTION on_round_start()
  base.on_round_start()
  weapon_statistics.clear() ; start_round_dump()
  corpse_list.clear()
  switch_phase(IN_PROGRESS)
  round = round + 1 ; round_start_time = server_time ; stamp the time string
  FOR EACH client
    clear its READY and VERY_VERY_DEAD flags
    its online-time baseline = server_time
  clear the disconnected pool
  clean the delayed event queue
  destroy every entity                       # SLS_Clear
  recreate the level's authored entities     # SLS_Default
  broadcast(ROUND_STARTED)
  request a synchronisation
```

**Invariants** — the order is load-bearing throughout. Flags are cleared **before** the world
is destroyed, so no handler observes a player marked dead against an entity that no longer
exists. The delayed events are discarded **before** the destruction, so no queued event
names an entity from the previous round. And the rebuild is a full teardown-and-recreate
rather than a reset, which is what guarantees a round starts from exactly the authored
state.

**Notes** — the two flags are cleared with a bitwise *addition* of the flag constants rather
than an OR. The two agree only because the flags occupy disjoint bits, which they do; a
rebuild should use a set union.

## `OnRoundEnd`

**Contract** — ends a round. Names the reason from a token table, stops any vote, switches to
the pending phase, broadcasts a round-over message carrying the reason, discards delayed
events, and then **tells the event queue to ignore everything from every client except the
server's own**.

**Notes** — the ignore switch is the interesting half. Between rounds the world is frozen but
the clients are still running and still generating events — a player who was mid-fall keeps
sending. Rather than validate each one against a dead round, the server stops listening
entirely, and listening resumes per client when that client reports it has started the next
round (see the event dispatch below). That is a clean way to draw a line across a
distributed system and a rebuild should keep it.

The reason token is a human-readable string on the wire, so the reason vocabulary is frozen
as text rather than as a number.

## `KillPlayer`

**Contract** — kills a player administratively, as opposed to by damage: used on disconnect,
on choosing to spectate, and by moderation. Refuses for an already-dead player and for a
non-actor. Notifies the mode's kill hook with the player as both killer and victim, stamps
the death time, broadcasts the kill message and a death event, and resets the player's
default items.

```text
FUNCTION kill_player(client, entity_id)
  object = client object for entity_id
  IF none OR not an actor: RETURN
  data = client record
  IF data exists AND its player is already permanently dead: RETURN
  IF data exists
    mode.on_player_kill_player(player, player, HIT, none, no weapon)   # self-kill
    player.clear_run_flag = false
  actor = object as an actor
  IF actor is not alive: RETURN                     # already dying by damage
  actor.set_death_time()
  send_player_killed_message(player, HIT, player, no weapon, none)
  broadcast event(DIE, destination = player) carrying the player and the client
  IF data exists: set_players_default_items(player)
  request a synchronisation
```

**Notes** — the kill is attributed to the victim themself with a plain hit type, because the
kill hook is the mode's only chance to adjust scores and the mode must be able to tell this
apart from a frag. Modes read the killer-equals-victim case as a suicide.

The two guards — the dead flag and the actor's own alive state — catch the same situation
from two directions, because the flag lags the actor by up to one update.

## `OnEvent`

**Contract** — the event dispatch for everything a multiplayer client can send: kills, hits,
readiness, paid spawns, the four vote messages, name changes, radio phrases, the game menu,
the started-the-round notification, and the two buy-menu bracket messages. Unknown types
fall through to the base game state.

**Notes** — two entries are not simply routing.

*The vote-start message is length-checked before it is read*, because it carries a free-form
string into a fixed buffer. That is the only untrusted variable-length field in this
dispatch and the check is what makes it safe.

*The started-the-round message carries the client's level name*, which is compared against
the server's. Only on a match is that client taken off the ignore list from the round end.
So the per-client resumption of event processing doubles as a map-agreement check: a client
on the wrong map is never re-enabled and is therefore inert. The reconnection that should
follow is present but commented out, so the client simply sits there — a real gap a rebuild
should close.

The two buy-menu messages are noted in the source as valid only from a dead player; nothing
enforces it here, and the modes that implement the hooks check for themselves.

## `RespawnPlayer` / `SpawnPlayer`

**Contract** — the respawn cycle, which is a **substitution of the client's server object**.
A dead actor is released to the server, queued as a corpse, and replaced by a spectator; a
spectator is destroyed and replaced by an actor. One flag skips the spectator stage and goes
straight back to playing.

```text
FUNCTION respawn_player(client, no_spectator)
  owner = the client's current server object
  IF owner is an actor
    allow its body to be removed ; queue it as a corpse
  IF owner is an actor AND NOT no_spectator
    spawn_player(client, "spectator")
  ELSE
    give the old object back to the server client
    IF owner is a spectator: broadcast destroy(owner)
    spawn_player(client, "mp_actor")

FUNCTION spawn_player(client, section)
  client.pass_updates = true
  mark the player permanently dead                 # see Notes
  entity = spawn_begin(section)
  entity.name = the client's name
  entity.flags = LOCAL + AS_PLAYER
  IF entity is an actor
    entity.team = player.team
    assign it a respawn point
    set its skin from the team's skin list
    clear the permanently-dead flag
    IF this is the player's first spawn: announce that he entered the game
    player.respawn_time = now
    statistics.on_player_spawned(player)
  ELSE                                              # spectator
    IF the dead actor's camera can be read: place the spectator there
    ELSE assign it a respawn point
  spawn_end(entity, client)
  player.game_id = the new object's id
  request a synchronisation
```

**Invariants** — the dead flag is **set at the top of the spawn and cleared only on the actor
path**. So a spectator is permanently-dead by construction, and that single flag is what the
whole game uses to mean "not currently playing". A rebuild must set it before the spawn, not
after, because the spawn broadcasts state.

The corpse is queued and released to the server *before* the replacement is spawned, so the
client never owns two objects.

**Notes** — a spectator inherits the dead actor's **camera** position and angles, not the
actor's own transform. The difference matters: the camera is where the player was looking
from, so the view does not jump at the moment of death. The angles are negated on the way
across, which is a coordinate-convention difference between the camera and the entity.

The two entity section names are literals here — a spectator and a multiplayer actor. They
are the only two things a multiplayer client can be.

## `OnPlayerDisconnect`

**Contract** — announce the departure, kill the player, release the body, queue it as a
corpse, then delegate.

**Notes** — the departing player's body stays in the world as a corpse rather than vanishing.
That is deliberate: a player who disconnects on being shot should not deny the kill or the
loot.

## `SetSkin`

**Contract** — assigns an entity's visual from the team's authored skin list, indexed by the
player's chosen skin, falling back to the team's first skin when the index is out of range.
Fails hard if the team has no skins loaded, and if the assembled name is 64 characters or
longer.

**Notes** — the length limit is a real constraint on the *data*, not on this code: the visual
name travels in the spawn record at a fixed width. A rebuild with a wider field can drop the
check, but the shipped spawn format cannot.

## `SetPlayersDefItems` / `ClearPlayerItems` / `ClearPlayerState`

**Contract** — rebuild a player's starting loadout: the team's authored default items, then a
per-rank substitution pass, then two boxes of base ammunition for each weapon that takes
any. `ClearPlayerState` additionally resets the kill, death and streak counters.

```text
FUNCTION set_players_default_items(player)
  clear the item list and the last-purchase amount
  IF player.team names a known team
    item list = that team's default items

  # promote items for every rank the player has reached, in order
  FOR rank IN 1 .. player.rank
    section = "rank_" + rank
    IF no such section: CONTINUE
    FOR EACH item IN the item list
      name = item_registry.name(item)
      key  = "def_item_repl_" + name
      IF the section has that key
        replacement = item_registry.index(section[key])
        IF the replacement is known: item = replacement

  # give ammunition for everything that fires
  FOR EACH item IN the item list (as it stood before this loop)
    name = item_registry.name(item)
    IF name is the knife: CONTINUE
    IF the item's section names an ammunition class
      append the FIRST ammunition class twice
```

**Invariants** — the rank substitution runs **once per rank in ascending order**, so an item
promoted at rank one can be promoted again at rank two. A rebuild that applies only the
player's current rank's section gets a different loadout.

**Notes** — the item registry maps a section name to a small index and back, and those
indices are what travel in the buy protocol. Note the registry lookup here masks the item
value to its **low byte** before asking for a name, while the bounds check compares the full
value: the two disagree above 255 items, which the shipped data does not reach. A rebuild
should pick one width.

Two boxes of ammunition, always, for every weapon. Not configurable, and the knife is
excluded by name.

## `SpawnWeapon4Actor` / `SetAmmoForWeapon` / `ChargeAmmo` / `ChargeGrenades` / `SpawnAmmoDifference`

**Contract** — spawn a purchased weapon into an actor's inventory, loaded. The weapon is
created parented to the actor with the requested attachment flags; its magazine is then
filled by **consuming ammunition entries from the player's own purchased item list**; any
remainder of a partly-consumed box is spawned as a separate ammunition item.

```text
FUNCTION charge_ammo(weapon, ammo_classes, player_items) -> leftover
  magazine = weapon.magazine_size ; weapon.loaded = 0
  FOR EACH class IN ammo_classes            # in the weapon's authored preference order
    id = item_registry.index(class)
    box = the class's configured box size
    WHILE player_items contains id
      weapon.ammo_type = this class's position
      remove one entry from player_items
      IF magazine - weapon.loaded <= box
        leftover = (class, box - (magazine - weapon.loaded))
        weapon.loaded = magazine
        BREAK                                # the magazine is full
      weapon.loaded = weapon.loaded + box
    IF weapon.loaded > 0: BREAK              # committed to this ammunition class
  IF weapon.loaded == 0
    weapon.ammo_type = 0
    IF the first class may be given free: weapon.loaded = magazine
```

**Invariants** — a magazine is loaded from **one** ammunition class only. The loop breaks out
as soon as any rounds have gone in, so a player carrying two compatible types gets the first
one his weapon prefers, not a mixture. That is a real gameplay rule and it is expressed only
by the placement of that break.

The leftover is a *single* pair, not a list, so only the last partly-consumed box is
returned as an item. Earlier boxes are consumed whole.

**Notes** — the free-ammunition fallback is a mode hook, disabled at this level. A mode that
enables it gives a full magazine to a player who bought a weapon but no rounds, which is what
stops the buy menu producing an unusable purchase.

Grenades are charged separately and differently: exactly **one** grenade of the first matching
type, out of at most four types, and the count and type are packed into a single byte. The
four-type limit is asserted against the weapon's configuration, so it is a constraint on the
data.

Ammunition preference order is the weapon's own configured list, so a rebuild must preserve
the order of that list — it is a gameplay decision expressed as data ordering.

## Rank and experience

**Contract** — `LoadRanks` reads a ladder of `rank_N` sections, each with a title, up to two
experience thresholds, a promotion bonus and a per-victim-rank experience multiplier table.
`Player_AddExperience` banks experience and promotes if the ladder and the mode allow it.
`Player_Check_Rank` compares banked plus pending experience against the *next* rank's first
threshold. `Player_Rank_Up` advances one rank, pays the bonus, and commits the pending
experience.

```text
FUNCTION on_player_hitted(packet)
  hitted = player for the hit entity ; hitter = player for the hitting entity
  IF either is unknown OR they are the same: RETURN
  IF the mode has no teams OR they are on different teams
    multiplier = ranks[hitter.rank].rank_diff_exp_bonus[hitted.rank]
    add_experience(hitter, damage * 100 * multiplier)
```

**Invariants** — experience is earned **by damage dealt, not by kills**, and scaled by how
much better the victim is than you. The multiplier table is indexed by the victim's rank
within the attacker's rank record, so beating a higher-ranked player pays more and farming a
beginner pays less. That two-dimensional table is the whole progression design.

Friendly fire earns nothing, and self-damage earns nothing.

**Notes** — the damage arrives as a fraction and is multiplied by a hundred here, which is the
fraction-to-percentage convention used throughout the creature code.

Two experience thresholds per rank are loaded and **only the first is ever read**. The second
is dead configuration with no consumer anywhere in this file. Its intended meaning — a
demotion threshold, perhaps — is not recoverable.

`rank_diff_exp_bonus` is read with an index guard that silently substitutes a multiplier of
one past the rank count, so a short table degrades to no bonus rather than failing. The guard
compares against the total rank count rather than the table's own length, which is the wrong
bound but harmless while the data is complete.

Promotion is gated on a mode-controlled flag, which the modes clear during warm-up: you
cannot rank up in a practice period.

## Money

**Contract** — `Player_AddMoney` adjusts a player's round balance, clamped below by the
team's configured floor and above by one million, and records the change for the statistics.
`Player_AddBonusMoney` additionally files an **itemised** entry — an amount, a reason, and a
kill count for streak bonuses — for the client to display.

```text
FUNCTION player_add_bonus_money(player, amount, reason, kills)
  IF amount is non-zero: player.bonus_money.append(amount, reason, kills)
  player_add_money(player, amount)
  player.money_added = player.money_added - amount      # see Notes
```

**Notes** — the last line undoes the running "money added" total that the plain add just
incremented. The two fields mean different things on the client: `money_added` is the
unattributed change ("you have more money") and the bonus list is the attributed one ("+500,
headshot"). A bonus must appear in exactly one of them, so it is added to the list and
subtracted back out of the total. A rebuild with one itemised ledger needs neither.

`UpdatePlayersMoney` pushes both to each client that has anything pending, once per update,
and clears them:

```text
FOR EACH client with money_added != 0 or a non-empty bonus list
  message(PLAYERS_MONEY_CHANGED)
    int32  current balance
    int32  unattributed change          # then cleared
    byte   number of bonus entries
    FOR EACH: int32 amount, byte reason, and a byte kill count only when the reason is a streak
  send to that client ; clear the bonus list
```

The conditional kill-count byte is the one variable-shape field in this layout and both ends
must agree on the streak reason value.

The upper clamp of one million has no derivation; it is a sanity bound.

## Voting

**Contract** — `OnVoteStart` accepts a command string from a player, matches its first word
against a table of seven known vote kinds, translates it into the console command that would
implement it, and broadcasts the proposal. `UpdateVote` tallies and either executes or
reports failure. `OnVoteYes` / `OnVoteNo` record one answer. `OnVoteStop` cancels.

```text
FUNCTION on_vote_start(command, sender)
  verb = first word ; params = the rest
  match verb against the table (restart, restart_fast, kick, ban, changemap,
                                changeweather, changegametype)
  IF matched
    IF that kind is disabled by the server's vote flags: RETURN
  ELSE IF the command does not begin with '$': RETURN      # see Notes

  voting_active = true ; vote_start_time = server_time
  build vote_command   (the console line to run if it passes)
  build voting_string  (what the players are shown)
  FOR EACH client: its answer = 1 if it is the proposer, else 2   # see Notes
  broadcast(VOTE_START) with the shown string, the proposer's name,
            and the vote duration in milliseconds
```

**Invariants** — the answer value **2 means "has not answered"**, 1 means yes and 0 means no.
The proposer is pre-set to yes. That three-valued encoding is what lets the tally distinguish
abstention from opposition, and the tally treats them very differently.

```text
FUNCTION update_vote()
  count over clients that are connected and not skipped:
    not_answered  (answer is neither 1 nor 0)
    agreed        (answer is 1)
    total
  against = total - agreed

  IF the vote window has NOT expired
    IF agreed > against + not_answered: succeed        # early success: unbeatable
    ELSE RETURN                                        # keep waiting
  ELSE
    IF participants-only counting: succeed if agreed / (not_answered + against) >= quota
    ELSE                          succeed if agreed / total >= quota

  voting_active = false
  broadcast(VOTE_END) with "succeeded" or "failed"
  IF it succeeded and this was a known command: execute vote_command on the console
```

**Notes** — the early-success test is the good part: a vote whose yes count already exceeds
every possible remaining no ends immediately rather than making everyone wait out the timer.
Note that it can only succeed early, never fail early.

The two expiry rules differ in whether abstainers count against the quota. With
participants-only counting off (the default), an abstainer is effectively a no, and the
default quota of 0.51 therefore needs an absolute majority of everyone present. With it on,
the denominator is the same set — the source's expression sums abstainers and opponents,
which equals the total — so the two formulas are in fact identical and the option changes
nothing. **That is almost certainly a bug**; the intent was clearly to divide by the number
who actually answered.

A command beginning with `$` bypasses the table entirely and is executed verbatim on the
console if it passes. That is a deliberate administrative escape hatch, and it means the vote
system can run **any** console command — a rebuild must gate it on the proposer's privileges,
which this code does not.

Kick and ban votes resolve the named player to a connection identifier at proposal time and
rewrite the command to address that identifier, falling back to the name-based command if
the player is not found. Resolving early is right: the player might be renamed before the
vote ends.

The ban vote's duration is parsed off the **end** of the parameter string by finding the last
space, which means a player whose name contains a space breaks the parse. Player names are
not otherwise restricted.

The vote duration is a configured number of **minutes**, defaulting to one.

## Bans

**Contract** — bans are keyed by the player's account digest, so they survive a reconnection
from a different address. Banning refuses for the server's own client and for anyone with
administrative rights. A ban can also be applied directly to a digest for a player who is not
connected.

**Notes** — keying on the account rather than the address is the decision; it is why banning
depends on the accounts seam rather than on the transport.

## `net_Export_State` / `SpectatorModes_Pack` / `SpectatorModes_UnPack`

**Contract** — the snapshot adds one byte: a bitmask of the spectator camera modes this server
permits (free flight, first person, look-at, free look, team camera). The same byte is read
from the session options.

**Notes** — the mask is packed by *camera mode enumeration value*, so the bit positions are
the camera enumeration's values and the two must not drift apart. The last one is packed at
the "maximum camera" sentinel's position, which is a bug waiting to happen if a camera mode
is ever added.

## Statistics

**Contract** — two dumps, both in the engine's configuration file format. The online dump is a
snapshot of who is connected, on what map, in what mode, with the map rotation. The round
dump is the same per-player data plus the round's start and end times and the weapon usage
breakdown. Per-player data includes rival, self and team kills, deaths, longest kill streak,
rank, artefacts, ping, money, online seconds, address, account digest, and four special-kill
counts.

The round dump is **asynchronous**: the server asks every client to upload its own statistics,
waits for all of them or for one maximum-ping interval, and then writes.

```text
FUNCTION dump_round_statistics_async()
  responses = one pending entry per client
  request_time = now
  broadcast(STATISTIC_UPDATE) carrying request_time

FUNCTION check_statistics_ready() -> bool
  IF no statistics period configured OR no request outstanding: RETURN true
  IF every client has answered OR request_time + max_ping < now
    write the dump ; close it ; clear the request
    RETURN true
  RETURN false
```

**Notes** — the timeout is one maximum client ping, which is far shorter than a real upload
would take. The effect is that the wait almost always times out rather than completing, and
the dump is written with whatever arrived. A rebuild should size the timeout to the work.

The special-kill counts are read out of the weapon statistics by **fixed index** — headshots,
backstabs, knife kills and "eye" kills at positions zero to three. Those indices are the
frozen meaning of that array and appear nowhere else as names.

The round dump's file name is timestamped at round start and the file is *removed* if the
round is restarted, so an abandoned round leaves nothing behind.

## `DestroyGameItem` / `RejectGameItem` / `DestroyAllPlayerItems`

**Contract** — destroy an entity outright; or detach it from its owner and leave it in the
world; or destroy everything an actor carries **except** his drop-bag, his knife and any
artefact.

**Notes** — the three exceptions are the rule. The bag is where a mode puts what a player
drops, the knife is the free starting weapon, and an artefact is an objective that must not
be deleted by a loadout change. Everything else is fair game when a purchase replaces a
loadout.

Rejecting a *grenade* first asks the grenade to drop itself — a primed grenade must land and
explode rather than simply appear on the ground, which is the difference between dying to a
grenade you were holding and not.

The destroy message is timestamped **two network latencies in the past**, which is how the
server backdates an event so that clients apply it at a moment they have already simulated
past rather than in their future. A rebuild with a different time-synchronisation model
needs an equivalent.

## `OnPlayerChangeName`

**Contract** — a rename request. Sanitises the proposed name, refuses outright on a public
server, then applies it: the client record, the account, the server object's display name,
and the weapon statistics are all updated, and every client is told the old and the new name
together.

**Notes** — the refusal on a public server is an anti-impersonation rule: on a listed server a
player's name is tied to his account. The message carries **both** names so that clients can
say "X is now Y" rather than silently swapping a label.

## `OnPlayerSpeechMessage` / `OnPlayerSelectSpectator` / `OnPlayerGameMenu`

**Contract** — a radio phrase is re-broadcast with the speaker's identifier prepended and its
three index bytes passed through untouched. The game-menu message dispatches to spectator,
team or skin selection, of which only the first is implemented here. Choosing to spectate
kills the player, sets the spectator flag, queues his body, and spawns a spectator.

**Notes** — the phrase indices are relayed **without validation**. Every receiving client
bounds-checks them itself (see
[`game_cl_capture_the_artefact_messages_menu.cpp`](game_cl_capture_the_artefact_messages_menu.cpp.md)),
which is the right place for it since the client owns the authored data — but it means a
malformed phrase message reaches every client rather than being stopped at the server.

## `RenewAllActorsHealth`

**Contract** — sets every living actor's health to full. Used at round transitions in modes
that heal between rounds.

**Notes** — it reaches the actors by looking up **client objects on the server**, which the
source itself flags as a hack that must be removed. In a single process that works; it is
exactly the server-object/client-object confusion the architecture is meant to avoid, and a
rebuild should write the authoritative record and let the update propagate.

## `OnNextMap` / `OnPrevMap` / `ReadOptions` / `Create`

**Contract** — map rotation cycles a list and issues a level change through the console,
guarded by a one-shot flag so a rotation cannot fire twice. `ReadOptions` reads the spectator
mask, the ping limit and the environment start time and rate from the session options.
`Create` loads the rank ladder and the ban list and disables promotion until a mode enables
it.

**Notes** — both rotation directions rotate the list and then read its new front, so the
rotation state *is* the list order. A rebuild storing an index instead must take care that
the list can change under it.

The environment start time is parsed as `hours:minutes` and converted into the game's own
calendar time at an arbitrary fixed date. The date is irrelevant; only the time of day is
used.

## `SvSendChatMessage` / `SetCanOpenBuyMenu` / `OnPlayerEnteredGame` / `GetTeamData` / `GetPosAngleFromActor`

**Contract** — a server-authored chat line under a fixed sender name; a message telling one
client its buy menu may open; an entered-the-game announcement; the team record lookup; and
reading an actor's camera position and angles.

**Notes** — the buy-menu-ready message reuses the **menu-closed** event as its carrier. That
is not a naming slip with no consequence: the client's handler for that event is what marks
the menu usable, so the two meanings are genuinely the same message. A rebuild should give it
its own name.
