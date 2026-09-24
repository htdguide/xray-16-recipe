# src/xrGame/game_sv_artefacthunt.cpp

> Artefact hunt on the authoritative side: one artefact cycling between authored points, hands and bases, with reinforcement waves, shielded bases, and a round that ends on a delivery count.

**Needs** — [`game_sv_artefacthunt.h`](game_sv_artefacthunt.h.md) · [`game_sv_teamdeathmatch.h`](game_sv_teamdeathmatch.h.md) · [`game_sv_artefacthunt_process_event.cpp`](game_sv_artefacthunt_process_event.cpp.md) · [`xrServer.h`](xrServer.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Artefact.h`](Artefact.h.md) · [`Inventory.h`](Inventory.h.md) · [`MPPlayersBag.h`](MPPlayersBag.h.md) · [`WeaponKnife.h`](WeaponKnife.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`ui/UIBuyWndShared.h`](ui/UIBuyWndShared.h.md) · [`debug_renderer.h`](debug_renderer.h.md) · [`Common/LevelGameDef.h`](../Common/LevelGameDef.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)

**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: session rules over the network and entity layers; the one format-facing edge is reading the level's authored point chunk

## Purpose

Team deathmatch already supplies two teams, friendly fire, a buy economy, ranks, team
balancing and round phases. Artefact hunt replaces the *reason a round ends* and adds one
object that the whole match revolves around.

Five things are this file's own:

1. **One artefact, cycling through four states.** It spawns at an authored point, lies on the
   field, is carried, and is delivered or expires. Every transition is a decision here, and
   the client's whole picture of the objective ([`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md))
   is derived from what this side broadcasts.
2. **Reinforcement waves instead of continuous respawn.** Dead players are held out until a
   wave, and may pay money to skip it.
3. **Shielded bases.** A player standing on his own base takes no damage and is teleported
   home when the round resets around him.
4. **Delivery scoring that pays the whole match.** Everyone on both teams gets money and
   experience from one delivery, on a scale that depends on which side they are.
5. **Respawn-point contention resolved by killing the occupant.** When every point is taken,
   the mode picks one held by an enemy and kills him to free it.

## State

```text
RECORD ArtefactHuntServer EXTENDS TeamDeathmatchServer
  artefact_state          : ArtefactState     # the four-state cycle below
  artefact_id             : int (16-bit)      # 0 = none exists
  artefact_bearer_id      : int (16-bit)      # 0 = nobody carries it
  team_in_possession      : int (8-bit)       # 0 = nobody
  af_bearer_menace_id     : int (16-bit)      # who last hurt the bearer; the assist credit
  artefact_spawn_time     : int (ms)          # deadline; 0 while the artefact exists
  artefact_remove_time    : int (ms)          # deadline; meaningful while it lies on the field
  artefacts_spawned_total : int (16-bit)      # see Notes: written, never incremented
  artefact_rpoints        : list<Point>       # authored spawn points, read once at level load
  artefact_chooser_random : random stream     # private, so point choice is independent
  next_reinforcement_time : int (ms)          # server clock; the next wave
  money_for_buy_spawn     : int (money)       # negative; default -10000
  no_lost_message         : bool              # suppress the "dropped" broadcast during a delivery
  artefact_was_taken      : bool              # this artefact has been picked up at least once
  artefact_was_dropped    : bool              # ...and has been dropped at least once
  artefact_brought_to_base: bool
  swap_bases              : bool              # mirror base ownership on the next entity creation

ENUM ArtefactState
  NONE            # before the first round; nothing has been decided
  NO_ARTEFACT     # none exists; the spawn deadline is running
  ON_FIELD        # it exists and is loose; the removal deadline is running
  IN_POSSESSION   # somebody is carrying it; neither deadline applies
```

Match-wide tunables, settable per match from the server's option string and defaulted here:

```text
artefact_respawn_delta = 30 seconds   # loose-to-next-spawn gap
artefacts_count        = 10           # deliveries needed to win; also the client's "frag limit"
artefact_stay_time     = 3 minutes    # how long a loose artefact survives; 0 = forever
reinforcement_time     = 15 seconds   # 0 = respawn immediately
                                      # -1 = respawn only when the artefact is delivered
bearer_cannot_sprint   = false
shielded_bases         = true
return_players_to_bases= true
```

Invariants:

- **`artefact_id` of zero means no artefact entity exists**, and it is the sentinel every
  predicate tests first. The state enumeration and this field must agree; the miss check
  below exists precisely because they can drift when an entity disappears unexpectedly.
- **Exactly one deadline is armed at a time.** Preparing for spawn arms the spawn deadline
  and clears the identifier; preparing for removal arms the removal deadline and *clears the
  spawn deadline to zero*. A rebuild that leaves both armed will spawn a second artefact
  while the first is still on the field.
- **`team_in_possession` and `artefact_bearer_id` are set and cleared together**, always
  followed by a synchronisation signal, because the client's map marker is built from the
  pair.

**Notes** — **two different clocks are used in one file and the split is not principled.**
The artefact's spawn and removal deadlines are measured against the *device* clock, while
the reinforcement wave and every respawn-point timeout are measured against the *server*
clock. They advance at the same rate but have different origins, so the two families of
deadline can never be compared. A rebuild should put everything on the server clock, which
is the one the protocol already speaks.

`artefacts_spawned_total` is reset in two places and **never incremented anywhere**. It is
read once, as an alternative to the "both teams are populated" test when deciding whether to
spawn — so that disjunct is dead, and the intent (once the first artefact has spawned, keep
spawning even if a team empties) is not realised. Flagged rather than reproduced.

## `Create`

**Contract** — called once when the mode is created for a level. Runs the base mode's
creation, then reads the level's authored point data and keeps the points marked as artefact
spawns for this game type. **Fails loudly if the level has none** — an artefact-hunt level
without artefact points cannot run. Arms the first spawn, reads the paid-spawn cost from
configuration, disallows rank-up by default, and seeds a private random stream.

```text
FUNCTION create(options)
  base.create(options)
  artefact_rpoints.clear()
  IF the level has an authored-point file
    FOR EACH point record in the point chunk
      read position, angles, team, type, game_type_mask
      IF type is "artefact spawn" AND game_type_mask includes artefact hunt
        artefact_rpoints.append(position, angles)
  IF artefact_rpoints is empty: FAIL WITH "no points to spawn artefact"

  artefact_prepare_for_spawn()
  artefact_id = 0 ; bearer = 0 ; team_in_possession = 0
  money_for_buy_spawn = config("artefacthunt_gamedata", "spawn_cost", default -10000)
  rank_up_allowed = false                 # see Notes
  seed artefact_chooser_random from a high-resolution counter
```

**Notes** — the point records carry a team, a type and a **game-type mask**, so one level's
point set serves every mode and each mode filters for its own. Artefact points ignore the
team field entirely: the artefact belongs to nobody.

Rank-up is switched **off** for the whole match and re-enabled only for the duration of a
delivery (see the delivery routine). That is how the mode makes rank advance a consequence
of scoring rather than of killing — a player cannot rank up mid-firefight.

The private random stream for choosing spawn points, seeded from a high-resolution counter
rather than from the shared stream, has no stated reason. The commented-out code it replaced
maintained a bag of unused points and excluded the previous one, so the *intent* was "do not
repeat the last point"; the live code is a plain uniform draw that can repeat. A rebuild
wanting the original behaviour must reinstate the bag; a rebuild wanting the current one
needs no separate stream at all.

## `OnRoundStart`

**Contract** — resets the round: clears the delayed-end flags, arms the reinforcement clock
to *now*, arms the artefact spawn, and respawns the level's items. When the reinforcement
mode is "only on delivery", every dead player is respawned immediately so that the round
starts with everyone alive.

**Notes** — the special case for the delivery-driven mode is load-bearing: in that mode there
is no wave to wait for, so without this a player who died in the previous round would start
the new one dead.

## The artefact cycle

### `Artefact_PrepareForSpawn`

**Contract** — forget the artefact, enter the no-artefact state, arm the spawn deadline at
now plus the respawn delta, clear the bearer and possession, and signal a snapshot.

### `Artefact_PrepareForRemove`

**Contract** — arm the removal deadline at now plus the stay time, and **disarm the spawn
deadline**. Called when an artefact appears on the field, whether freshly spawned or dropped.

```text
FUNCTION artefact_prepare_for_spawn()
  artefact_id = 0 ; artefact_state = NO_ARTEFACT
  artefact_spawn_time = now_device + respawn_delta_seconds * 1000
  bearer = 0 ; menace = 0 ; team_in_possession = 0
  signal_synchronize()

FUNCTION artefact_prepare_for_remove()
  artefact_remove_time = now_device + stay_time_minutes * 60000
  artefact_spawn_time  = 0
```

**Notes** — the respawn delta is configured in **seconds** and the stay time in **minutes**.
Two units in two adjacent settings, and the scale factors are the only place that says so. A
rebuild should normalise both to one unit at the configuration boundary.

### `Artefact_NeedToSpawn`

**Contract** — part of the per-update pass. Spawns the artefact when none exists, the spawn
deadline has passed, and spawning is allowed. Reports whether it acted, so the caller can
stop the pass.

```text
FUNCTION artefact_need_to_spawn() -> acted
  IF state is ON_FIELD or IN_POSSESSION: RETURN false
  IF artefact_id is not 0:               RETURN false
  IF artefact_spawn_time >= now_device:  RETURN false
  IF NOT artefact_spawn_allowed():       RETURN false     # see the dead disjunct in State
  artefact_spawn_time = 0
  spawn_artefact()
  RETURN true
```

### `Artefact_NeedToRemove`

**Contract** — removes a loose artefact whose stay time has expired. A configured stay time
of zero means "never expires" and disables the whole branch. A carried artefact never
expires.

**Notes** — the stay time exists so that a loose artefact nobody is contesting does not sit
in a corner for the whole match; it despawns and reappears somewhere else after the respawn
delta. Zero disabling the rule rather than meaning "immediately" is the sentinel convention
used throughout the mode.

### `Artefact_MissCheck`

**Contract** — the safety net. If the mode believes an artefact exists but the server has no
such entity, it re-arms the spawn from scratch. Reports whether it acted.

**Notes** — this is recovery, not a rule, and it is the honest admission that the state
enumeration can drift from reality: an artefact can be destroyed by a path that does not run
through this mode. A rebuild with a single owner for the artefact's lifetime can drop it,
but should keep the *decision* — the objective must never be permanently missing.

### `SpawnArtefact`

**Contract** — creates the artefact server object from the section named in the mode's
configuration, at a randomly chosen authored point, owned by the server itself. Broadcasts
the spawned event, enters the on-field state, arms the removal deadline, starts the level's
anomaly set if anomalies are enabled, and clears the taken and dropped flags. Does nothing
at all if no artefact section is configured.

```text
FUNCTION spawn_artefact()
  IF no "artefact" line in the mode's configuration: RETURN
  entity = spawn_begin(that section)
  entity.flags = SPAWN_OBJECT_LOCAL                 # created here, not replicated from a level record
  assign_artefact_rpoint(entity)
  artefact_id = spawn_end(entity, owner = the server's own client).id
  broadcast(GAME_EVENT_ARTEFACT_SPAWNED)            # no payload
  artefact_state = ON_FIELD
  artefact_prepare_for_remove()
  signal_synchronize()
  IF anomalies are enabled: start_anomalies()
  artefact_was_taken = false ; artefact_was_dropped = false
```

**Notes** — the anomaly set is started **with the artefact**, not with the round. So the
level's hazards appear when there is something to contest and are the environmental cost of
going for it. The set is named by the mode (`artefacthunt_game_anomaly_sets`), which is how
a level can ship different hazard layouts per mode.

Returning silently when no artefact section is configured means a misconfigured match runs
forever with no objective and no diagnostic. A rebuild should fail at creation instead.

### `RemoveArtefact`

**Contract** — broadcasts the destroyed event naming the artefact, destroys the entity, and
arms the next spawn. Safe when no artefact exists — it still re-arms.

### `Assign_Artefact_RPoint`

**Contract** — places a spawning artefact at one authored point chosen uniformly at random
from the mode's private stream.

### `ArtefactSpawn_Allowed`

**Contract** — true only when **both** teams have at least one non-spectating, connected,
ready player. Counts players by team over the whole client list.

**Notes** — the objective does not appear until there is a contest. Without this a single
player joining an empty server would farm deliveries unopposed. Note that it counts
*present* players, not living ones, despite the internal name suggesting otherwise.

## Ownership transfer

### `OnTouch`

**Contract** — intercepts a creature picking something up. When an actor touches the
artefact, it records the bearer and the possessing team, enters the possession state,
broadcasts the taken event and — on the **first** pickup of this artefact — pays an
experience bonus to every member of the taker's team. Weapons that are buyable items are
accepted without comment; anything else falls through to the base mode.

```text
FUNCTION on_touch(who, what) -> accepted
  IF who is an actor
    IF what is the artefact
      bearer = who ; menace = 0 ; team_in_possession = who.team
      signal_synchronize() ; artefact_state = IN_POSSESSION
      broadcast(GAME_EVENT_ARTEFACT_TAKEN, taker.game_id, taker.team)
      IF NOT artefact_was_taken
        artefact_was_taken = true
        FOR EACH connected, playing member of the taker's team
          add experience("af_first_take_all")
      RETURN accepted
    IF what is a weapon that appears in the buy list: RETURN accepted
  RETURN base.on_touch(who, what)
```

**Invariants** — the first-pickup bonus fires **once per artefact**, not once per round, and
the flag is cleared when a new artefact spawns. So a team is rewarded for reaching each
artefact first, which is the behaviour that makes a fresh spawn worth racing for.

**Notes** — clearing the "menace" identifier on pickup discards any assist credit accumulated
against the previous bearer. Correct: the assist is credit for softening up *this* carrier.

The buyable-weapon clause short-circuits the base mode's pickup rules for anything in the buy
list, so a player can always pick up equipment of a kind he could have bought. It has nothing
to do with the objective and is simply grafted here.

### `OnDetach`

**Contract** — an actor loses the artefact. Clears the bearer and possession, returns to the
on-field state, marks it dropped, broadcasts the dropped event *unless* the drop is part of
a delivery, and arms the removal deadline afresh.

**Invariants** — the removal deadline is re-armed on every drop, so an artefact that is
repeatedly picked up and dropped never expires while it is being contested. That is the
intended reading of "stay time": time spent *unattended*.

**Notes** — the suppression flag is the interesting part. A delivery destroys the artefact
while it is still in the scorer's hands, which reaches this handler as a drop; broadcasting
"player dropped the artefact" immediately after "player scored" would be nonsense. The
delivery routine therefore raises the flag around the destroy and lowers it again. A rebuild
with an explicit "consumed" transition needs no flag.

## Base occupancy and delivery

### `OnObjectEnterTeamBase` / `OnObjectLeaveTeamBase`

**Contract** — an actor entering or leaving **its own** team's base zone sets or clears the
on-base flag and signals a snapshot. On entry, the actor's carried items are scanned for the
artefact; finding it is the delivery.

```text
FUNCTION on_object_enter_team_base(who, zone_team)
  IF who is not an actor OR who.team is not zone_team: RETURN
  who.player.set_flag(ON_BASE)
  signal_synchronize()
  IF who's carried items contain artefact_id
    on_artefact_on_base(who's client)
    statistics.record_artefact_brought(who.player)
```

**Invariants** — entry into an *enemy* base is ignored entirely, in both handlers. The
on-base flag therefore means "on my own base", which is what the shield rule and the buy rule
both need; carrying the artefact into the enemy base does nothing.

**Notes** — delivery is detected by **scanning the actor's children for the artefact
identifier**, not by an event on the artefact. That means a player who has the artefact
stowed rather than held still scores, which is the intended generosity.

### `OnArtefactOnBase`

**Contract** — the scoring routine, and the largest single decision in the mode. Resets the
field, pays everybody, increments the team score, destroys the artefact, restores the
scorer, and arms the next spawn.

```text
FUNCTION on_artefact_on_base(scorer_client)
  IF reinforcement is delivery-driven OR players are returned to bases
    move_all_alive_players()                    # teleport survivors to spawn points
  IF reinforcement is timed OR delivery-driven
    respawn_all_not_alive_players()
  respawn_level_items()

  scorer = the player record for that client ; IF none: RETURN

  rank_up_allowed = true                        # only for the duration of this payout
  pay scorer:   money(target_succeed), experience("target_succeed")
  scorer.artefact_count = scorer.artefact_count + 1
  FOR EACH other connected, playing player
    IF same team as scorer
      money(target_succeed_all), experience("target_succeed_all")
    ELSE
      money(target_failed)
      IF the artefact was never dropped
        multiply their pending experience by "target_failed_all_mul"
    finalise their experience
  rank_up_allowed = false

  team_score[scorer.team - 1] = team_score[scorer.team - 1] + 1
  no_lost_message = true ; destroy the artefact entity ; no_lost_message = false
  broadcast(GAME_EVENT_ARTEFACT_ONBASE, scorer.game_id, scorer.team)
  restore the scorer to full health and full stamina
  signal_synchronize() ; ask everyone to refresh statistics
  artefact_prepare_for_spawn()
```

**Invariants** — rank-up is enabled for exactly the span of the payout and disabled again.
Every experience award in the match therefore either happens here or cannot cause a rank
change. A rebuild must preserve the bracket, not merely the awards.

**Notes** — three rules are buried in the payout and each is a real design decision.

*Everyone is paid, including the losers.* The losing team receives a consolation amount, so a
delivery does not leave them unable to re-equip — the economy is kept from running away from
the team that is behind.

*An artefact delivered without ever being dropped costs the losers extra.* Their pending
experience is multiplied by a configured factor for a clean run. That is the only penalty in
the game that scales with how well the *other* side played, and the multiplier is the
mechanism that makes an uncontested delivery sting.

*Delivering resets the field.* Depending on the mode's settings, survivors are teleported to
spawn points and the dead are brought back, so a delivery starts a fresh engagement rather
than leaving the scoring team spread across the map with a positional advantage. That is
what makes the mode a sequence of rounds inside a round.

The scorer is healed to full and given full stamina. Delivering is meant to be survivable.

The artefact destruction here is composed by hand rather than through the usual event
generator — the message is begun explicitly and stamped with the device clock. Incidental,
and it is why the suppression flag has to bracket exactly these lines.

## Reinforcement waves

### `RespawnAllNotAlivePlayers`

**Contract** — respawns every connected, playing, permanently dead non-spectator, gives each
a fresh weapon loadout and a clear-run bonus, signals a snapshot, and re-arms the next wave
at now plus the interval.

**Notes** — re-arming the wave inside the respawn routine, rather than only in the update
loop, means **any** cause of a mass respawn resets the wave timer. So a delivery that
respawns everyone also postpones the next wave by a full interval, which is intended: nobody
should be respawned twice a second apart.

### `OnPlayerReady` (the wave gate)

**Contract** — a dead player asking to spawn is refused while the round is in progress, the
reinforcement mode is not "immediate", the player has not paid, and warm-up is over.
Otherwise the base mode handles it.

```text
FUNCTION on_player_ready(client)
  IF round is in progress AND client currently controls a spectator
     AND reinforcement is not immediate AND client has not paid
     AND warm-up is over
    RETURN                                   # silently refuse; wait for the wave
  base.on_player_ready(client)
```

**Invariants** — the warm-up escape is what lets everyone spawn freely before the round
proper starts; the paid flag is what lets one player out of the wave. Both are exceptions to
the same rule and both must be tested here, or the wave becomes inescapable.

### `OnPlayerBuySpawn`

**Contract** — the player paid to skip the wave. Accepted only from a permanently dead player
who has not already paid. Sets the paid flag, charges the (negative) cost, and immediately
runs the ready path. If the player is alive afterwards — the spawn succeeded — the flag is
cleared again so that the next death costs afresh.

```text
FUNCTION on_player_buy_spawn(client)
  IF client is not permanently dead: RETURN
  IF client has already paid:        RETURN
  client.paid_for_spawn = true
  add_money(client, money_for_buy_spawn)     # negative
  on_player_ready(client)
  IF client is no longer permanently dead
    client.paid_for_spawn = false            # consumed
```

**Notes** — the flag is both the permission and the receipt, cleared by observing that the
spawn worked rather than by the spawn path reporting success. A rebuild with a return value
should use it; the observe-afterwards shape leaves the flag set if the spawn silently fails,
and the player is then never charged again.

The money is added rather than subtracted because the configured cost is stored **negative**.
The client's affordability test does the same, so the sign convention is shared across the
wire ([`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md)).

## Respawn-point assignment

### `assign_rp_tmp`

**Contract** — private. Given a candidate point list, produces the indices of the points that
are free, and separately the indices of points occupied by an **enemy** together with that
enemy's client. A point counts as occupied when any living player stands within 0.4 units of
it, or when it is administratively blocked. In forced mode, proximity is ignored and only
administrative blocks disqualify a point; if that still yields nothing, every point is
returned.

```text
FUNCTION assign_rp_tmp(who, points, free_out, enemy_points_out, enemy_clients_out, forced) -> any_free
  free_out.clear()
  FOR EACH index, point IN points
    blocked = false
    FOR EACH connected client with a living, spawned actor
      IF distance(point, that actor) <= 0.4 units AND NOT forced
        blocked = true
        IF that actor's team differs from who's team AND teams exist
          enemy_points_out.append(index) ; enemy_clients_out.append(that client)
    IF blocked OR point.administratively_blocked: CONTINUE
    free_out.append(index)
  IF forced AND free_out is empty
    free_out = every index                       # last resort: overlap somebody
  RETURN free_out is not empty
```

**Notes** — the flag's name says "forced find" and its effect is to **disable** the proximity
test, so the two callers read backwards: the first call passes true (ignore proximity, just
avoid administrative blocks) and the second passes false (do the full proximity test and
collect enemies). A rebuild should name it for what it does.

0.4 units is roughly a body radius — the distance at which two actors would be spawned
inside one another.

### `assign_RP`

**Contract** — chooses a spawn point for an actor. Spectators and non-actors fall through to
the base mode. For an actor, the candidate list is that actor's **team's** point list. If no
point is free, one held by an enemy is chosen and **that enemy is killed** to free it.
Otherwise a free point is chosen uniformly. Fails loudly if neither yields anything.

```text
FUNCTION assign_rp(entity, who)
  IF entity is a spectator or not an actor: base.assign_rp(entity, who) ; RETURN
  points = rpoints[who.team]
  IF NOT assign_rp_tmp(who, points, free, enemy_points, enemy_clients, forced = true)
    assign_rp_tmp(who, points, free, enemy_points, enemy_clients, forced = false)

  IF free is empty AND enemy_points is not empty
    pick = a uniform choice among enemy_points
    set_rp(entity, points[pick])
    send GAME_EVENT_PLAYER_KILL naming the occupying enemy      # free the point
  ELSE
    IF free is empty: FAIL WITH "no free respawn points"
    set_rp(entity, points[uniform choice among free])
```

**Invariants** — spawn points are **per team**, indexed by the actor's team, which is what
makes bases defensible: a team always spawns on its own side.

**Notes** — killing the occupant is the most surprising rule in the mode. When one team's
spawn area is entirely overrun, the spawning player is placed on an enemy and that enemy dies
outright. That is the anti-spawn-camping measure: camping a spawn area gets the camper killed
rather than the spawner. A rebuild must keep the *decision* — a spawn may never be blocked by
an enemy body — even if it chooses a gentler mechanism.

The kill is delivered as an ordinary game event naming the victim as both actor and target,
so the victim kills himself. The camper's killer is therefore nobody, and the spawning player
receives no credit — which is right, since he did nothing.

### `SetRP` / `CheckRPUnblock`

**Contract** — `SetRP` writes the point's position and angles into the entity, marks the point
administratively blocked, records who blocked it and when, and adds it to the blocked list.
`CheckRPUnblock` releases blocked points: a block older than one second expires, and so does
a block whose owner has vanished or has moved to within 0.4 units of the point.

```text
FUNCTION check_rp_unblock()
  FOR EACH point IN blocked_list
    IF NOT point.blocked:                       release
    ELSE IF point.block_time + 1000 ms < now:   release
    ELSE
      owner = the entity that blocked it
      IF owner is none OR distance(point, owner) <= 0.4 units: release
```

**Invariants** — the block exists to bridge the gap between *deciding* where an entity will
appear and it *actually being there*. It is released the moment the entity is observed at the
point, and unconditionally after one second so that a spawn that never completes cannot
block a point forever.

**Notes** — the release-when-close condition reads inverted and is not: the owner arriving
**at** the point is what makes the reservation unnecessary, because from then on ordinary
proximity blocking takes over. One second is the assumed worst case for a spawn to complete.

### `RP_2_Use`

**Contract** — which point list an entity spawns from: its team's. Non-actors get list zero.

## Player movement replication

### `MoveAllAlivePlayers`

**Contract** — teleports every living player to a spawn point (unless he is already on his
base), heals him to full, stops his motion, gives him full stamina, and broadcasts one
message carrying every moved player's new placement.

```text
FUNCTION move_all_alive_players()
  moved = 0 ; payload = empty
  FOR EACH connected, living, spawned player
    IF NOT player.on_base: assign_rp(player's entity, player)
    heal to full ; move the client object to the entity's placement ; stop any motion
    send that client a full-stamina event
    payload.write_int16(entity.id)
    payload.write_vector(entity.position) ; payload.write_vector(entity.angles)
    moved = moved + 1
    player.pass_updates = false ; player.last_move_update = server_now
  IF moved == 0: RETURN
  broadcast M_MOVE_PLAYERS { byte moved, then the payload }
```

**Invariants** — a player standing on his own base keeps his position. His base is already
where he would be sent, and teleporting a defender off his own feet at the moment of a
delivery would be both pointless and disorienting.

**Notes** — the wire layout is worth stating once: **one count byte followed by that many
records of (entity identifier, position, angles)**, all in one reliable broadcast. Batching
matters — the alternative is one message per player at exactly the moment the match is most
congested.

Each moved player is marked as not having acknowledged the move, with a timestamp. That is
the input to the retry below.

### `UpdatePlayersNotSendedMoveRespond` / `ReplicatePlayersStateToPlayer`

**Contract** — once per update, finds **one** client that was moved more than a second ago and
has not acknowledged it, and re-sends it the full placement of every living player in the same
message shape. Marks it acknowledged and re-stamps it.

**Notes** — one client per update, not all of them. That is deliberate rate limiting: a
teleport storm that loses several acknowledgements is repaired over several updates rather
than by a burst of full-state messages. The repair is also *complete* rather than
differential — the client receives everyone's placement, not just its own — because a client
that missed a move message has an unknown amount of stale state.

## Round end

### `CheckForTeamElimination`

**Contract** — only run in the delivery-driven reinforcement mode. When one team has no living
players — and is not simply empty — the other scores a point, every member of the winning
team is paid a wipe-out bonus, the phase switches to the corresponding eliminated phase, and
the artefact is removed.

```text
FUNCTION check_for_team_elimination()
  winner = 0
  IF team 1 has no living players: winner = 2
  ELSE IF team 2 has no living players: winner = 1
  IF winner == 0: RETURN
  team_score[winner - 1] = team_score[winner - 1] + 1
  FOR EACH connected, playing member of the winning team: add_money(rivals_wiped_out)
  phase = (winner == 1) ? TEAM2_ELIMINATED : TEAM1_ELIMINATED
  begin the delayed team-eliminated sequence
  remove_artefact()
```

**Invariants** — `CheckAlivePlayersInTeam` returns **true for an empty team**. A team with no
players at all has not been eliminated; without that, an unbalanced server would score a
point every update. That single line is the difference between a working mode and an infinite
score.

**Notes** — the phase constants are crossed: team 1 winning switches to the *team 2
eliminated* phase, because the phase names the loser. Easy to invert in a rebuild.

Elimination scores like a delivery — both are worth one point toward the artefact count — so
wiping the enemy out is an alternative win condition inside the delivery-driven mode.

### `CheckForTeamWin`

**Contract** — ends the round when either team's score reaches the artefact count, or when the
time limit expires. On a tie at the time limit the round does **not** end unless the server is
configured to skip waiting for a winner, in which case team 1 is declared the winner.

```text
FUNCTION check_for_team_win()
  IF team_score[0] >= artefacts_count: winner = 1
  ELSE IF team_score[1] >= artefacts_count: winner = 2
  ELSE
    IF no time limit: RETURN
    IF server_now - round_start <= time_limit_minutes * 60000: RETURN
    IF the scores are equal
      IF NOT skip_winner_waiting: RETURN          # play on until someone scores
      winner = 1                                  # see Notes
    ELSE winner = the higher score
  record the team score ; switch to that team's scoring phase
  begin the delayed round end, reason "artefact limit"
```

**Notes** — the tie-break awards the round to **team 1** rather than resolving it, and only
when the server has been told not to wait. Arbitrary, undocumented, and the kind of thing a
rebuild should replace with an explicit draw.

### `OnTimelimitExceed`

**Contract** — the base mode's time-limit hook. Does nothing on a tie; otherwise scores the
round to the higher team and ends it with the time-limit reason.

**Notes** — **this duplicates the time-limit branch above and disagrees with it.** Both paths
can end the same round on the same condition, with different end reasons, and they differ on
a tie: this one always plays on, while the other can award the round to team 1. Which runs
first is an ordering accident. A rebuild should have exactly one time-limit path.

## Combat rules

### `GetKillResult` / `OnKillResult`

**Contract** — promotes an ordinary kill to a *critical* kill when the victim is the artefact
bearer, on both the teammate and the rival paths. A critical teammate kill counts as a team
kill, pays the killer the target-team bounty, and is reported as not counting as a frag; a
critical rival kill increments the rival-kill counters and pays the target-rival bounty,
multiplied when the killer was invincible at the time.

**Invariants** — killing the bearer is a distinct, more valuable event than killing anyone
else, and the distinction is made *before* the base mode sees it. Everything downstream — the
scoreboard, the bounty, the frag accounting — keys off the promoted result.

**Notes** — a critical teammate kill still **pays** the killer, at the enemy-team rate, while
also counting against him as a team kill. That reads like a bug: the money branch does not
distinguish killing your own bearer from killing theirs. It is what the shipped code does.

### `OnGiveBonus` / `OnPlayerKillPlayer` / `OnPlayerHitPlayer`

**Contract** — assist credit. Any enemy hit on the bearer records that hitter as the current
"menace". Killing the menace afterwards earns an assist bonus. The record is cleared when the
menace dies, when the artefact changes hands, and when it is dropped.

```text
FUNCTION on_player_hit_player(hitter, hitted)
  base.on_player_hit_player(...)
  IF either is permanently dead OR they are teammates: RETURN
  IF hitted is the artefact bearer: menace = hitter
```

**Notes** — the model rewards *protecting the carrier*: whoever is hurting your bearer becomes
a named target, and killing him pays. The record holds one person — the most recent attacker —
so a bearer under fire from several enemies generates credit for only one of them at a time.

A critical rival kill is passed to the bonus path as an ordinary rival kill, so the promotion
affects money and frags but not the bonus table.

### `check_Player_for_Invincibility` / `OnPlayerHitPlayer_Case`

**Contract** — when shielded bases are enabled, a player standing on his own base is flagged
invincible, and any hit on him other than a physical strike has its damage and impulse zeroed.
Otherwise the base mode's invincibility rules apply.

**Invariants** — the shield is applied in **two** places: as a flag the client mirrors, and as
damage nullification at the point of a hit. Both are needed — the flag drives presentation and
the nullification is the actual rule.

**Notes** — melee is deliberately exempt. A player can be attacked at his own base only by
someone who has walked into it, which is precisely the risk the shield is meant to price. The
shield makes a base a refuge, not a fortress.

### `Check_ForClearRun`

**Contract** — pays a clear-run bonus on every respawn.

**Notes** — **the condition this bonus was meant to have is commented out.** The disabled code
tested and then set a per-player flag, so the bonus was once paid at most once. As shipped it
is paid on every respawn, which makes it an unconditional respawn allowance rather than a
bonus for anything. The original intent is not recoverable from the source; the name suggests
a reward for a life without a death, which the disabled code does not implement either.

## Snapshot export

### `net_Export_State`

**Contract** — appends this mode's state to the full snapshot, after team deathmatch's. The
reinforcement deadline is present **only** when the reinforcement interval is positive.

```text
FUNCTION net_export_state(packet, to_client)
  base.net_export_state(packet, to_client)
  packet.write_byte (artefacts_count)              # the target, not a running total
  packet.write_int16(artefact_bearer_id)
  packet.write_byte (team_in_possession)
  packet.write_int16(artefact_id)
  packet.write_byte (bearer_cannot_sprint)
  packet.write_int32(reinforcement_time)           # SECONDS; 0 immediate, -1 on delivery
  IF reinforcement_time > 0
    packet.write_int32(next_reinforcement_time - server_now)   # MILLISECONDS, relative
```

**Invariants** — this layout is frozen against the decoder in
[`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md), field for field, including the
conditional last field. Both ends must agree on the condition or every subsequent field of the
snapshot is misread.

**Notes** — the interval is sent in **seconds** and the time-to-next-wave in **milliseconds**,
in adjacent fields of one message. The client converts the second into absolute server time on
arrival and keeps the first as seconds. Nothing justifies the mismatch; a rebuild should send
both in milliseconds and will change both ends together.

The subtraction that produces the remaining time is unsigned and is **not** clamped, so a
deadline that has already passed sends a value near the top of the range rather than zero. The
client displays it as a countdown of many days. The window is one update wide and the wave
re-arms immediately, which is why it is rarely seen.

## Supporting rules

### `Player_Check_Rank`

**Contract** — in addition to the base mode's rank requirements, a player may not advance until
his delivered-artefact count reaches the next rank's second term.

**Notes** — this is what makes rank in artefact hunt reflect *objective play* rather than
kills. A player who never delivers cannot outrank one who does, whatever his frag count, and
it works with the rank-up bracket around the delivery payout: both halves must hold.

### `OnCreate`

**Contract** — notices entities as they are created. An artefact becomes *the* artefact. A team
base zone has its team mirrored when the base-swap flag is set.

**Notes** — adopting any artefact that appears means an artefact placed in the level by any
other route silently becomes the objective. Simple, and it is also how the mode recovers after
the miss check re-arms.

The base mirror is `3 - team`, the standard two-team flip, and it is applied at creation
because a base zone's team cannot usefully be changed once the level is live.

### `SwapTeams`

**Contract** — swaps the two teams' players by forcing the base mode's automatic swap on for
one call and restoring the previous setting.

**Notes** — the code that would have swapped the **spawn points and the bases** instead is
commented out, so the mode swaps *players* between teams rather than swapping the geography. A
rebuild has both options and should pick one deliberately; the disabled version is the one the
base-mirror flag above exists to serve, and with it disabled that flag is never set.

### `LoadTeams` / `ReadOptions` / `GetAnomalySetBaseName` / `WriteGameState`

**Contract** — load the mode's price list and three team sections (neutral, and one per team),
failing loudly if the price section is missing; read the four match options from the server's
option string; name the mode's anomaly sets; and record the artefact limit in the round's
written result.

**Notes** — reading the options **forces the frag limit to zero**. Artefact hunt inherits team
deathmatch's frag-limit end condition and must disable it, or a match would end on frags
before anyone delivered anything. That one assignment is the whole of the override.

A negative reinforcement option of any magnitude is normalised to exactly -1, so the sentinel
cannot be reached by accident with an arbitrary negative number.

### `ConsoleCommands_Create` / `ConsoleCommands_Clear`

**Contract** — both empty. The mode registers no console commands of its own; its four tunables
are reached through the option string and through the base mode's variables.

### `OnRender`

**Contract** — debug builds only. Draws a vertical line and a flattened ellipse at each
authored artefact point when the respawn-point debug flag is set.

**Notes** — the only way to see where a level's artefact points actually are; the points are
authored in a tool and never otherwise visible.
