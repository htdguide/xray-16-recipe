# src/xrGame/game_sv_capture_the_artefact.cpp

> The authoritative side of capture the artefact: two artefacts that must be carried home while your own sits at its base, a delivery that resets the field, an asynchronous buy cycle for the dead, and a round that ends on a delivery count.

**Needs** — [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md) · [`game_sv_capture_the_artefact_buy_event.cpp`](game_sv_capture_the_artefact_buy_event.cpp.md) · [`game_sv_capture_the_artefact_myteam_impl.cpp`](game_sv_capture_the_artefact_myteam_impl.cpp.md) · [`game_sv_capture_the_artefact_process_event.cpp`](game_sv_capture_the_artefact_process_event.cpp.md) · [`game_sv_mp.h`](game_sv_mp.h.md) · [`game_base.h`](game_base.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrServerEntities/clsid_game.h`](../xrServerEntities/clsid_game.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Artefact.h`](Artefact.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`HudItem.h`](HudItem.h.md) · [`Missile.h`](Missile.h.md) · [`eatable_item_object.h`](eatable_item_object.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`UIGameCTA.h`](UIGameCTA.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`ui/UIBuyWndShared.h`](ui/UIBuyWndShared.h.md) · [`Common/LevelGameDef.h`](../Common/LevelGameDef.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md) · [`game_sv_capture_the_artefact_buy_event.cpp`](game_sv_capture_the_artefact_buy_event.cpp.md) · [`game_sv_capture_the_artefact_myteam_impl.cpp`](game_sv_capture_the_artefact_myteam_impl.cpp.md)
**Tier floor** — T2: session rules over the network and entity layers; the one format-facing edge is reading the level's authored point chunk

## Purpose

Two teams, two artefacts, two bases. Each team's artefact rests at its own home point. You
score by carrying the *enemy's* artefact to *your* home point — but only while your own
artefact is sitting there. You defend by touching your own displaced artefact, which sends it
home. That is the whole mode, and it is the classic capture-the-flag rule set expressed in
this engine's vocabulary.

Six concerns live here, and only the first two are about the objective:

1. **The artefact cycle** — taken, carried, dropped, returned, delivered — with a home-point
   proximity test standing in for every one of those events.
2. **Delivery, which resets the field.** Scoring teleports the survivors to spawn points,
   heals them, and brings the dead back. A match is a sequence of engagements separated by
   deliveries, not one continuous fight.
3. **An asynchronous buy cycle.** A dead player shops while he waits; his respawn is held
   until he closes the menu, and the wave that would have taken him is remembered.
4. **Timed invincibility** on respawn, held per client in a deadline table rather than derived
   from the respawn time.
5. **Team membership** — selection, automatic balancing, automatic swapping between rounds —
   re-implemented here rather than inherited (see
   [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md) on why).
6. **A second anomaly rotation**, structurally different from deathmatch's and sharing its
   console variables.

**None of it runs in a shipped build**: the transport seam ships its null filling by default
(see [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)),
so a match cannot be started without editing the build description.

The client's picture of all this is in
[`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md), and every field of
the two messages below is frozen against its decoder.

## State

The session record is in
[`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md); the per-team artefact
record is in
[`game_sv_capture_the_artefact_myteam_impl.cpp`](game_sv_capture_the_artefact_myteam_impl.cpp.md).
What this file adds is a set of **process-wide settings**, each also a console variable, and
that is itself the invariant:

```text
invincibility_time      = 5 seconds
artefact_returning_time = 45 seconds      # a loose artefact's patience
activated_artefact_ret  = 0               # changes what touching your own artefact does
player_scores_delay     = 3 seconds       # the end-of-round dwell
artefact_base_radius    = 1.0             # "at the base point" means within this
rank_up_arts_divisor    = 1
```

Alongside them the mode reads **three other modes' settings** — deathmatch's anomaly, PDA-hunt,
warm-up and time-limit variables, team deathmatch's balance, swap, friendly-fire and team-kill
variables, and artefact hunt's score limit and reinforcement interval. They are file-scope
values shared across the whole process, which is sound only because exactly one session runs
at a time, and it means an operator's friendly-fire setting reaches both mode chains at once.

## `Update` — the phase machine

**Contract** — the per-update pass, dispatching on the round phase. Three phases: a lobby, a
running round, and a scoreboard dwell.

```text
FUNCTION update()
  base.update()
  SWITCH phase
    IN_PROGRESS:
      check_for_artefact_delivering()       # before the clock is sampled; position-based
      current_time = server_clock
      check_for_warmup(current_time)
      reset_timeout_invincibility(current_time)
      check_anomaly_update(current_time)
      check_for_artefact_returning(current_time)
      IF spectator_mode: follow an active player and caption the view
      IF current_time has passed next_reinforcement_time
        respawn_dead_players()
        next_reinforcement_time = current_time + reinforcement_interval
      IF check_for_round_end()
        dump the round statistics asynchronously
        switch to PLAYER_SCORES
        next_reinforcement_time = current_time + scores_delay   # REUSED FIELD
        signal_synchronize()

    PENDING:
      check_statistics_ready()
      IF the round has not started AND the level's game configuration has finished loading
        IF all players are ready
          IF a map rotation has a next map AND the auto-swap policy permits: go there
          ELSE start the round
        ELSE IF a fast restart is pending: start the round

    PLAYER_SCORES:
      current_time = server_clock
      IF current_time has passed next_reinforcement_time: end the round
```

**Invariants** — the wave field is **reused as the scores-phase dwell deadline** the moment the
round ends. The two are never live together, but a rebuild should name them separately.

The pending branch does not sample the clock, so the reinforcement countdown exported to a
client joining the lobby is computed against whatever the last running round left behind.
Harmless — nothing is counting down in a lobby — but a rebuild should read the clock where it
is used rather than caching it in a field.

**Notes** — **the pending phase has no timeout.** Deathmatch starts its round after ten seconds
whatever anyone has pressed; this mode starts only when every player is ready or an operator
issues a fast restart. Combined with the readiness rule below — which does *not* count a
still-loading client as ready — one client that never finishes loading stalls the server
indefinitely. The source even marks the spot, noting that such a player ought to be voted
out. A rebuild should adopt deathmatch's timeout.

The map-rotation condition is written as two branches that differ only in which half of the
auto-swap policy they check, and together they say: rotate the map unless auto-swap is on and
the teams have not yet been swapped on this map. That is the rule — **each map is played
twice, once from each side, before the rotation advances** — and it is worth stating plainly
because the code does not.

## `CheckForRoundStart` / `CheckForAllPlayersReady`

**Contract** — the round starts only on a fast restart, or when every connected player counts
as ready. A player counts if he has signalled ready, is on the spectators' team, or is marked
skip. The server's own client is exempt when it is not yet network-ready; **every other client
that is not yet network-ready counts as not ready.**

```text
FUNCTION check_for_all_players_ready() -> bool
  IF there is no server client: RETURN false
  ready = 0
  FOR EACH client
    IF it has no player record: CONTINUE
    IF it is not network-ready AND it is the server's own: CONTINUE   # not counted at all
    IF its team is the spectators' team: ready = ready + 1 ; CONTINUE
    IF it has the READY flag OR is marked skip: ready = ready + 1
  RETURN ready == player_count AND ready != 0
```

**Invariants** — a spectator is always ready, because he is not going to play and must not
hold the lobby.

**Notes** — the line that would have counted a still-loading client as ready is present and
disabled, with a note saying such a player should be kicked by a vote instead. The effect is
the stall described above. The sibling modes made the opposite choice
([`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md)) and accept that a player may join
mid-round; that is the better trade.

## `CheckForRoundEnd`

**Contract** — ends the round when either team reaches the score limit, or when the time limit
has expired **and the scores differ**. Suppressed entirely during warm-up.

```text
FUNCTION check_for_round_end() -> bool
  IF warming up: RETURN false
  IF either team's score >= score_limit
    round_end_reason = ARTEFACT_LIMIT ; RETURN true
  IF no time limit: RETURN false
  IF server_clock - round_start <= time_limit minutes: RETURN false
  IF the two scores differ
    round_end_reason = TIME_LIMIT ; RETURN true
  RETURN false
```

**Invariants** — **a tied clock does not end the round; it plays on until somebody delivers.**
Same overtime rule as deathmatch, and here it has no escape hatch at all — the debug override
that lets the other modes resolve a tie is not consulted. A match between two evenly matched
teams runs until one of them scores.

## `OnRoundStart`

**Contract** — begins a round: forgets the previous round's anomaly identifiers, resets the
spectator camera, arms warm-up unless this is a fast restart, runs the base round start (which
destroys the world and rebuilds it from the authored spawn records), applies the team swap and
balance policies, puts every client into a fresh spectator body with its starting money and
default items, zeroes both scores, arms the first wave, spawns both artefacts, loads and
starts an anomaly set, respawns the level's items, and discards every buy-menu state.

```text
FUNCTION on_round_start()
  anomaly_ids.clear()                         # see Invariants
  reset the spectator camera
  in_warmup = false ; warmup_deadline = 0
  IF NOT fast_restart AND warmup_time > 0
    warmup_deadline = server_clock + warmup_time seconds ; in_warmup = true

  was_fast_restart = fast_restart             # the base start clears the flag
  base.on_round_start()
  round_started = true

  IF the round did not end by force and was not a fast restart
    IF auto_swap:    swap_teams()
    IF auto_balance: balance_teams()
  IF the round ended by a full restart: teams_swapped = false

  FOR EACH client with a player record
    clear the record and its item list
    back-date its death time by about a second
    give it the team's default items and starting money
    spawn it a spectator body

  both scores = 0
  next_reinforcement_time = was_fast_restart ? now : now + reinforcement_interval / 5
  respawn_artefacts() ; load_anomaly_set() ; restart_random_anomaly()
  respawn every world item and every level item
  discard every buy-menu state ; signal_synchronize()
```

**Invariants** — **the anomaly identifier table must be cleared before the world is rebuilt.**
It maps zone names to the entities that answered to them last round, and those entities are
about to be destroyed. The base start re-runs the creation hook for every zone, which refills
it. Getting this order wrong sends zone-state changes to identifiers that name nothing, and
the level's hazards silently stop working.

**A fast restart never re-arms warm-up**, which is what makes warm-up terminate: the warm-up
deadline expiring issues a fast restart, and that restart starts a round with no warm-up.

**Notes** — the first wave after a normal round start comes at **a fifth of the interval**, not
a full one. Nothing explains the fraction. The intent is plainly that players who begin the
round in a spectator body should not wait a full wave to enter, and a rebuild should express
that as "the first wave is immediate" rather than as a divisor.

Every player is put into a spectator body rather than into the world, and entry into play is a
separate act — the same arrangement as deathmatch, and it is what gives the client its
pre-round team, skin and buy window.

The teams are swapped and rebalanced **only** when the previous round ended normally. A round
restarted by an operator keeps its sides, which is what makes a restart a do-over rather than
a new match.

## `OnRoundEnd`

**Contract** — destroys every playing client's actor body, spawns each a spectator, delegates
to the base round end, and clears the ready and permanently-dead flags from everyone.

**Invariants** — the bodies are destroyed explicitly rather than left as corpses, because the
next thing that happens is the lobby and a corpse in a lobby is a corpse in the next round's
opening seconds.

**Notes** — the two flags are cleared with a bitwise *addition* of the flag constants rather
than a union. The two agree only because the flags occupy disjoint bits, which they do. A
rebuild should use a set union; the same pattern appears in
[`game_sv_mp.cpp`](game_sv_mp.cpp.md).

## The objective

### `OnTouch`

**Contract** — the ownership veto, and the mode's central rule. A player touching **an
artefact** is doing one of three things depending on whose it is and where it is; a player
touching anything else falls through to the item rules.

```text
FUNCTION on_touch(who, target, forced) -> allowed
  actor = who, if it is a multiplayer actor ; IF not: RETURN allowed
  team_of_artefact = the team whose artefact has identifier `target`
  IF there is none: RETURN on_touch_item(actor, target)

  IF team_of_artefact == actor.team                     # --- MY OWN artefact ---
    IF the artefact is AT its home point: RETURN denied  # it is where it belongs
    IF activated_artefact_ret is 0                       # the default
      move_artefact_to_point(artefact, its home point)   # returned, by touching it
      pay the toucher the team's KILL_RIVAL amount
      broadcast(ARTEFACT_TAKEN, team_of_artefact, toucher)
      RETURN denied                                      # he does not carry it
    IF the toucher already carries an artefact: RETURN denied
    record him as the artefact's owner ; RETURN allowed   # he carries it home himself

  # --- the ENEMY's artefact ---
  IF the toucher already carries an artefact: RETURN denied
  record him as the artefact's owner
  broadcast(ARTEFACT_TAKEN, team_of_artefact, toucher)
  RETURN allowed
```

**Invariants** — **one artefact per player.** Both pickup paths refuse a player who already
owns one, which is what stops a single player carrying both and ending the match alone.

**The same broadcast means two different things**, and the *artefact's owning team* is what
separates them: a take message naming your own team is a **return**, naming the other team is
a **capture**. The client derives both from that one field (see
[`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md)), which is why the
team must be the artefact's and not the player's.

**Notes** — the `activated_artefact_ret` setting flips the defensive rule between two quite
different games. At its default of zero, **touching your displaced artefact teleports it home
instantly** and you are paid for it — a defender's job is to reach the artefact, not to escort
it. Non-zero, you must *pick it up and carry it*, and the activation path below becomes the
way to send it home. The shipped default is the first. A rebuild should treat these as two
modes rather than as a number.

The return bounty is read from the team's **kill-a-rival** money key. That is a reuse of an
unrelated configuration entry and it means the price of returning your artefact cannot be
tuned independently of the price of a kill. A rebuild should give the return its own key; the
team record already has `target_team` sitting unused for exactly this.

### `OnActivate`

**Contract** — the use veto. A player may activate an artefact only if it is **his own team's**;
doing so records the activation time and him as the last activator. Anything else is refused.

**Notes** — activation is only meaningful when the touch rule above is in its carry-it-home
variant: the returning check sees an activated artefact and teleports it home at once, paying
the last activator. So activation is "put it back from here" and the rule it implements is
that a defender who has fought his way to the artefact should not also have to walk it home
under fire.

This is the only mode that overrides the activation veto at all.

### `CheckForArtefactReturning`

**Contract** — run every update. For each team whose artefact is unowned and away from its home
point: an *activated* artefact goes home immediately and pays its activator; an ordinary loose
artefact goes home once it has lain untouched for the configured returning time.

```text
FUNCTION check_for_artefact_returning(now)
  FOR EACH team
    IF the artefact has an owner: CONTINUE          # being carried; not loose
    IF the artefact is at its home point: CONTINUE
    IF the artefact is activated
      move_artefact_to_point(artefact, home point) ; deactivate it
      activator = the last activator, if he still exists
      IF activator exists
        pay him the KILL_RIVAL amount of his team
        broadcast(ARTEFACT_TAKEN, activator.team, activator)
      CONTINUE
    IF free_since is unset OR now - free_since >= returning_time
      move_artefact_to_point(artefact, home point)
      free_since = now
```

**Invariants** — **`free_since` is stamped on drop and cleared on pickup**, so the returning
clock measures time spent *loose*, not time spent away from home. An artefact repeatedly
picked up and dropped in a contested corridor never returns itself, which is the intended
reading: the timer exists to recover an artefact nobody is fighting over, not to punish a
fight.

**Notes** — the unset case moves the artefact home *immediately*. It is reachable only in the
narrow window where an owner disappeared without a drop, and it is the right recovery: an
artefact with no owner and no drop timestamp is lost, not loose.

The delay before an activated artefact returns is present as commented-out code and the live
path returns it at once. So the activation is instantaneous and the configured delay is not
applied anywhere. Whether an escort period was intended is not recoverable.

### `CheckForArtefactDelivering` / `ActorDeliverArtefactOnBase`

**Contract** — detection and scoring. A carrier delivers when he is standing at **his own**
team's home point, **his own** team's artefact is at that point, and **nobody is carrying his
own team's artefact.** Delivering pays the scorer and his team, increments the team score,
sends the carried artefact home, and resets the field.

```text
FUNCTION check_for_artefact_delivering()
  FOR EACH team whose artefact has an owner
    carrier = that owner ; my_team = carrier's team
    IF my_team's artefact has an owner:                      CONTINUE   # mine is out
    IF my_team's artefact is NOT at my_team's home point:     CONTINUE  # mine is displaced
    IF carrier is within the base radius of my_team's home point
      deliver(carrier, my_team, the artefact's team)

FUNCTION deliver(carrier, scoring_team, artefact_team)
  drop the carried artefact AT the artefact team's home point   # returned, not consumed
  broadcast(ARTEFACT_ONBASE, scoring_team, carrier)
  pay carrier:  money(target_succeed), experience("target_succeed")
  carrier.artefact_count = carrier.artefact_count + 1
  scoring_team.score = scoring_team.score + 1

  allow rank-up                                    # only for the span of this payout
  FOR EACH other connected, playing player
    IF on the scoring team: money(target_succeed_all), experience("target_succeed_all")
    ELSE                    finalise their pending experience with nothing added
  disallow rank-up

  signal_synchronize() ; record the delivery ; ask everyone to refresh statistics
  start_new_round()
```

**Invariants** — **your own artefact must be home for a delivery to count.** That single
condition is what makes the mode a contest rather than a race: a team that sends everyone
forward cannot score, because the enemy will have taken their artefact. It is the rule a
rebuilder is most likely to drop, and dropping it produces a mode where both sides simply run
past each other.

**Rank-up is enabled for exactly the span of the payout and disabled again.** Combined with
the rank check below, promotion can only land at a delivery — never mid-fight, where a changed
loadout would be disruptive.

**Notes** — the delivered artefact is **returned to its owner's base, not destroyed.** Compare
artefact hunt, which consumes its single artefact and spawns another. The difference is the
whole reason this mode reads as capture-the-flag and that one as a scramble.

The losing side's players are handed zero experience, which looks pointless and is not: the
call finalises whatever experience they had pending, committing it at a moment the mode
controls. That is the same idiom artefact hunt uses.

Everybody's placement resets after a score — see below — so a delivery is a round boundary in
everything but name.

### `StartNewRound` / `PrepareClientForNewRound` / `MoveLifeActors`

**Contract** — the post-delivery reset. Every living player is given full stamina and
reassigned a respawn point; their new placements are broadcast in one message; everyone is
healed to full; the dead are brought back; the level's items respawn.

```text
FUNCTION start_new_round()
  FOR EACH client: prepare_client_for_new_round(client)
  move_life_actors()
  renew every actor's health
  respawn_dead_players()
  respawn the level's items

FUNCTION prepare_client_for_new_round(client)
  IF the player is permanently dead: RETURN          # the respawn below handles him
  send that client a full-stamina event
  assign_rp(his entity, his player record)           # writes a new placement

FUNCTION move_life_actors()
  payload = empty ; count = 0
  FOR EACH connected, network-ready, non-skipped, living client
    payload.write_int16(entity.id)
    payload.write_vector(entity.position) ; payload.write_vector(entity.angles)
    count = count + 1
  broadcast M_MOVE_PLAYERS { byte count, then the payload }
```

**Invariants** — the placement is **assigned first and broadcast second**, as two passes over
the clients. They cannot be merged: the message must carry the new positions, not a mixture of
old and new.

**Notes** — the wire layout is one count byte followed by that many records of (entity
identifier, position, angles), in one reliable broadcast. Batching matters — the alternative
is one message per player at exactly the moment the match is most congested. Artefact hunt
sends the identical message for the identical reason, and additionally retries it for clients
that do not acknowledge; this mode does not, so a lost teleport message leaves one client with
a stale world until its next ordinary update corrects it.

Unlike artefact hunt, a player standing on his base is **not** exempted from the teleport —
everyone is moved. Since a delivery happens at a base, the scorer is among those relocated.

### `ReSpawnArtefacts` / `MoveArtefactToPoint`

**Contract** — one artefact per team, created from the class named in that team's configuration
section, owned by the server, placed at the team's home point. Moving one writes the position
into the authoritative record, stops any activation on the live object, moves the live object,
and **broadcasts the move explicitly**.

**Invariants** — the move must be broadcast rather than left to the ordinary update stream. A
resting artefact stops sending position updates — it is asleep — so a client that is not told
about the teleport keeps rendering it where it was. The message carries a count, then
(identifier, position) per artefact, and the count is always one here.

**Notes** — the artefacts are created with the flag that marks an entity as **made by this
server**, not replicated from a level record, which is what keeps them out of the level's
authored spawn set and therefore out of the round-start teardown's rebuild.

### `LoadArtefactRPoints`

**Contract** — reads the level's authored point data at creation and keeps, for each team, the
one point marked as an artefact spawn for this game type. Fails loudly if either team has none.

```text
FUNCTION load_artefact_rpoints()
  IF the level has an authored-point file
    FOR EACH point record in the point chunk
      read position, angles, team, type, game_type_mask
      team = team - 1                                    # see Invariants
      IF game_type_mask does not include this mode: CONTINUE
      IF type is "artefact spawn": that team's home point = (position, angles)
  FOR EACH team: FAIL WITH "no home point" IF it was never set
```

**Invariants** — **the authored team number is one-based and this mode's is zero-based**, so
every point's team is decremented on the way in. The level editor numbers the green team 1 and
the blue team 2. The same adjustment is made for the base-entry events in
[`game_sv_capture_the_artefact_process_event.cpp`](game_sv_capture_the_artefact_process_event.cpp.md)
and is *not* made in the artefact-hunt equivalent. The level data is the authority and it is
one-based; a rebuild should convert once, where the level is loaded, and let every mode see
zero-based indices.

The game-type mask is what lets one level's point set serve every mode: each mode filters the
same records for its own bit.

## Respawn and protection

### `RespawnPlayer`

**Contract** — runs the base respawn, pays the team's per-respawn amount, raises the
invincibility flag and records its deadline, and spawns the player's backpack onto the new body.

**Invariants** — the backpack is spawned on every respawn. It is what a corpse's items are
transferred into on death, so a body without one drops nothing.

The invincibility deadline is held in a **table keyed by client**, not derived from the
player's respawn time. That is a genuine difference from deathmatch, which recomputes the
expiry from the respawn stamp on every sweep, and it is what lets this mode expire protection
early on leaving a base without disturbing the respawn stamp.

### `ResetTimeoutInvincibility` / `ResetInvincibility`

**Contract** — the sweep: every deadline that has passed and is not already spent clears its
client's invincibility and is marked spent. One synchronisation is forced if anything changed.
The single-client form is also called when a player leaves his own base.

```text
FUNCTION reset_timeout_invincibility(now)
  changed = false
  FOR EACH (client, deadline) IN invincibility_deadlines
    IF deadline is non-zero AND now >= deadline
      changed = reset_invincibility(client)
      deadline = 0                        # spent, not removed
  IF changed: signal_synchronize()
```

**Invariants** — a spent deadline is **zeroed rather than removed**, because removing while
walking the table is what the loop cannot do. The entry is removed only on disconnect. A
rebuild with a safe removal does not need the sentinel.

**Notes** — only the *last* client to be reset in a pass sets the changed flag, since each
iteration overwrites it. With a single synchronisation per pass the effect is the same
whenever any reset happens to be last, and one is missed whenever a successful reset is
followed by an unsuccessful one. The synchronisation arrives on the next tick regardless, so
the consequence is a one-tick delay, not a lost state. A rebuild should accumulate the flag.

Unlike deathmatch, **firing does not forfeit protection here** — the mode does not override the
fire hook at all, and the base's is empty. Protection expires on its timer or on leaving your
base, and a protected player can shoot for five seconds. That is the single largest rule
difference from deathmatch and it is not obviously intended.

### `RespawnDeadPlayers` / `RespawnClient` / `OnPlayerReady`

**Contract** — a wave respawns every permanently dead non-spectator, *except* those with the
buy menu open, who are marked ready-to-spawn instead and respawned when they close it. A
respawn gives the player his purchased loadout if he bought since dying, and the rank's
default loadout if he did not.

```text
FUNCTION respawn_dead_players()
  FOR EACH client with a player record and a body
    IF the player is a spectator: CONTINUE
    IF he is in the buy menu: mark him READY_TO_SPAWN
    ELSE                      respawn_client(client)

FUNCTION respawn_client(client)
  IF the player is not permanently dead: RETURN
  respawn_player(client, no spectator body)
  IF he has not bought anything since dying
    clear his item list ; refill it with the team's rank-adjusted default items
  spawn his weapons ; mark him READY ; signal_synchronize()

FUNCTION on_player_ready(client)
  IF phase is PENDING
    toggle the READY flag ; signal_synchronize() ; RETURN
  IF phase is IN_PROGRESS
    IF this is the server's own client and spectator mode is on
      follow the next active player ; RETURN
    IF the player is not permanently dead, or is marked skip: RETURN
    respawn_player(client, allow spectator body)
    IF the new body is an actor
      IF he has not bought anything since dying: reset him to the default items
      spawn his weapons
      pay him the team's clear-run bonus
```

**Invariants** — **the "has bought since dying" flag is what stops a respawn overwriting a
purchase.** A dead player's purchase is recorded but not spawned (see
[`game_sv_capture_the_artefact_buy_event.cpp`](game_sv_capture_the_artefact_buy_event.cpp.md));
if the respawn refilled his item list from defaults first, the purchase would be silently
discarded.

**Notes** — the ready path in progress **respawns a dead player immediately**, with no wave
check of any kind. The wave gate that makes waves feel like waves lives entirely on the
client. See
[`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md) — this is a rule on the
untrusted end and a rebuild must move it.

The clear-run bonus is paid **here and again on death** (below), so a player who respawns by
pressing the key collects it twice per life while one respawned by a wave collects it once.
Artefact hunt has the same bonus with its condition commented out; here it has no condition to
begin with. In both modes it is an unconditional respawn allowance rather than a bonus for
anything, and the name is left over from something that no longer exists.

## Kill accounting

### `GetKillResult` / `OnKillResult`

**Contract** — classification and consequence. Four outcomes: nothing, a self-kill, a kill of a
team-mate, and a kill of a rival. A self-kill costs the configured amount; a rival kill pays,
reduced if the killer was protected at the time; a team kill pays the (negative) team-kill
amount, counts against the killer, and **disconnects him once he reaches the team-kill limit**,
if punishment is enabled.

```text
FUNCTION on_kill_result(result, killer, victim) -> may_bonus
  team = team data for the killer's team
  SWITCH result
    NONE:      RETURN false
    SELF:      killer.self_kills = killer.self_kills + 1
               killer.best_streak = 0                      # see Notes
               money(kill_self) ; RETURN false
    RIVAL:     killer.rival_kills = killer.rival_kills + 1
               killer.streak = killer.streak + 1
               killer.best_streak = max(killer.streak, killer.best_streak)
               amount = kill_rival
               IF killer is currently invincible: amount = amount * invincible_modifier
               money(amount) ; RETURN true
    TEAMMATE:  money(kill_team)                             # normally negative
               killer.team_kills = killer.team_kills + 1
               killer.best_streak = 0
               IF punishment is on AND killer.team_kills >= team_kill_limit
                 disconnect him with a localised reason
               RETURN false
```

**Invariants** — a kill made under spawn protection pays a **reduced** reward, halved by
default. That is the anti-spawn-camping rule: the protection exists so you can leave your
spawn, not so you can trade shots for free. It is a money penalty only — the frag still counts.

Only a rival kill earns situational bonuses.

**Notes** — **a self-kill and a team kill zero the killer's best-streak *record* rather than his
running streak.** Deathmatch zeroes the running streak, on the victim, where it belongs. The
effect here is that suiciding wipes your scoreboard achievement while leaving your streak
bonus climbing — precisely backwards. The running streak *is* zeroed, correctly, in the
victim-processing path below, so the two halves of the same rule live in two places and one of
them is wrong.

The team-kill punishment resolves the offending player by walking the client list looking for
the record it already holds, and refuses to act on the server's own client. The disconnect
reason comes from the localised string table, so the message the player sees is data.

### `OnGiveBonus`

**Contract** — situational bonuses on a rival kill: experience and money for a headshot, an eye
shot, a backstab or a knife kill, plus a money bonus named for the killer's current streak
length. The whole payout is bracketed by permitting rank-up.

```text
FUNCTION give_bonus(result, killer, victim, kill_type, special, weapon)
  IF killer is none: RETURN
  allow rank-up
  IF result is RIVAL AND kill_type is HIT
    IF special is HEADSHOT / EYESHOT / BACKSTAB
      experience(cfg exp <name>) ; bonus_money(cfg money <name>, <name>)
    ELSE IF the weapon is a knife
      experience(cfg exp "knife_kill") ; bonus_money(cfg money "knife_kill", KNIFEKILL)
  IF result is RIVAL AND killer.streak > 0
    bonus_money(cfg money "<streak>_kill_in_row", KIR, streak)
  disallow rank-up
```

**Invariants** — the streak bonus looks up a configuration key **built from the streak number**,
so the shipped data decides how long a streak must be before it pays: an absent key yields
zero. The reward curve is entirely in data with no length compiled in.

**Notes** — this mode pays **experience as well as money** for a backstab and for a knife kill;
deathmatch pays money only for both. The difference is not explained anywhere and is easy to
miss because the two routines are otherwise line-for-line identical. A rebuild should pick one
table and share it — the duplication is the cause, not the symptom.

Only hit kills earn special bonuses. Bleeding and radiation deaths pay the base rival amount
and nothing else.

### `OnPlayerKillPlayer` / `ProcessPlayerDeath`

**Contract** — the death entry point: process the victim, classify, apply, pay bonuses if
allowed, record in the weapon statistics, and force a synchronisation. Processing the victim
marks him permanently dead, clears his ready flag, counts the death, zeroes his running streak,
pays him the clear-run bonus, clears his "bought since dying" flag, and **drops the artefact if
he was carrying one**.

```text
FUNCTION process_player_death(victim)
  victim.set_permanently_dead ; victim.clear_ready
  victim.deaths = victim.deaths + 1 ; victim.streak = 0
  pay victim the team's clear_run_bonus
  victim's "bought since dying" flag = false
  IF victim owns a team artefact: drop_artefact(victim, that artefact)
  record the death in the weapon statistics
```

**Invariants** — **the artefact is dropped where the carrier died**, with no position override,
so the objective lands on his body. The returning timer then starts and the artefact goes home
by itself if nobody contests it.

The clear-run bonus is paid on **death**, and the drop is detected by searching the two teams
for one whose artefact names this player as owner rather than by asking the player what he
carries. Two entries, so the search is trivial; a rebuild with more teams should index it.

### `OnPlayerHitted` / `OnPlayerHitPlayer` / `OnPlayerHitPlayer_Case`

**Contract** — `OnPlayerHitted` brackets the base's damage-experience path with rank-up
permitted, so experience earned by damage can promote. `OnPlayerHitPlayer` reads the hit
description out of the message, stamps the attacker's identity, lets the mode adjust it,
records the last hitter if any damage survives, and writes the adjusted description back.
The adjustment applies friendly fire and then the protection shield.

```text
FUNCTION adjust_hit(hitter, hitted, hit)
  IF hit.type is a physical strike: RETURN          # neither rule applies
  IF hitter and hitted are on the same team AND are not the same player
    hit.power = hit.power * friendly_fire_modifier
    hit.impulse = hit.impulse * (modifier > 1 ? modifier : 1)
  IF hitted is invincible
    hit.power = 0 ; hit.impulse = 0
```

**Invariants** — **the hit is rewritten in place in the message**, so every later reader sees the
adjusted values. That is what makes both rules authoritative rather than advisory.

Protection is applied **after** friendly fire and overrides it, which is the only sensible
order: a protected player takes nothing from anyone.

**Notes** — friendly fire scales damage but **never reduces impulse**; the impulse multiplier is
clamped at one from below. So a teammate's shot still shoves you the full amount even when it
barely scratches. That is deliberate — the shove is feedback that tells you a teammate is
shooting you — and a rebuild that scales both loses the signal.

Neither rule applies to a physical strike, so a protected or friendly player can still be
shoved by an explosion or a vehicle and is never an immovable obstacle in a doorway.

`OnPlayerHitted`'s rank-up bracket is this mode's own: artefact hunt disables the same path
entirely. So in capture the artefact a player can be promoted by dealing damage, and in
artefact hunt only by delivering. Both are deliberate and they differ.

## `Player_Check_Rank`

**Contract** — on top of the ladder's experience threshold, a player may not advance beyond the
**leading team's score** scaled by a divisor. Experience that would have promoted him is parked
exactly at the threshold instead.

```text
FUNCTION player_check_rank(player) -> may_promote
  IF player is already at the top rank: RETURN false
  next_threshold = ranks[player.rank + 1].terms[0]
  IF player.banked + player.pending < next_threshold: RETURN false
  IF player.rank + 1 > max(green_score, blue_score) * rank_up_divisor
    player.pending = next_threshold - player.banked     # park it at the threshold
    RETURN false
  RETURN true
```

**Invariants** — **the gate is the match's leading score, not the player's own contribution.**
Nobody outranks the match. With the shipped divisor of one, no player reaches rank one until
somebody has delivered once, rank two until somebody has delivered twice, and so on.

**Parking the pending experience at the threshold is what stops it accumulating.** Without it a
player held back by the score gate would bank arbitrarily much and then jump several ranks the
instant a delivery landed. With it, he is always exactly one delivery away. That is the whole
purpose of the clamp and it is the subtlest line in the mode.

**Notes** — artefact hunt implements the same idea against the *player's own* delivered count
rather than the match's leading score. Two readings of "rank should reflect objective play",
and the difference matters: here a player who never touches an artefact still ranks up as long
as somebody does.

## Teams

### `OnPlayerSelectTeam` / `OnPlayerChangeTeam`

**Contract** — a team request is applied, acknowledged to the requesting client with a menu
response, and broadcast through a synchronisation. A request for team "any" picks the team with
the fewest players. **Changing team costs a life.**

```text
FUNCTION on_player_change_team(player, requested)
  IF requested is "any"
    team = the team with the fewest players ; that team's count = count + 1
  ELSE team = requested
  player.team = team
  IF player.money is zero: set it to the team's starting amount
  signal_synchronize()

# the caller then, if the team actually changed:
kill the player                      # the new side takes effect immediately
```

**Invariants** — the kill is conditional on the team having **actually changed**, so a player
re-selecting his own team is not punished for it.

Money is reset only when it is zero, so switching sides mid-match does not refund you. A
player who has spent everything is topped up to his new team's starting amount, which is the
generosity that keeps a swapped player playable.

**Notes** — the "fewest players" count is a field on the team record that is incremented here
and **never decremented anywhere**. So the automatic assignment drifts: after enough joins and
leaves it sends everyone to whichever team happened to receive fewer explicit "any" requests,
regardless of who is actually playing. The balancing pass below counts players properly and
repairs it between rounds; within a round it does not. A rebuild should count the clients.

### `BalanceTeams` / `SwapTeams`

**Contract** — balancing counts the non-spectating, non-skipped players on each side and moves
**half the difference**, lowest score first, from the larger team to the smaller. Swapping
exchanges every player's side and records that this map has been swapped.

```text
FUNCTION balance_teams()
  count the playing members of each team
  IF the counts are equal: RETURN
  larger, smaller = the two teams by count
  to_move = (larger_count - smaller_count) / 2
  REPEAT to_move times
    victim = the member of `larger` with the lowest frag count
    victim.team = smaller
```

**Invariants** — **half the difference is exactly the number that levels the teams**, since each
move changes the gap by two. An integer division rounding down leaves a gap of one, which is
the best achievable with an odd total.

**The lowest-scoring player is moved.** Moving your best player would swing the match; moving
the one who is contributing least is the smallest possible disturbance. It is also the
harshest — the player being moved is the one already having a bad match — and a rebuild might
reasonably prefer the most recent joiner.

**Notes** — the search is re-run from scratch for every player moved, which is the simple
correct thing since each move changes the candidate set. It also asserts that a candidate was
found, so a balance pass on a team that has emptied under it stops the server. In practice the
count was taken a moment earlier and cannot have changed.

Both run only between rounds, and only when the previous round ended normally.

### `LoadTeamData` / `Money_SetStart` / `LoadSkinsForTeam` / `LoadDefItemsForTeam`

**Contract** — each team's configuration section supplies its skins, its default items, the name
of the artefact class it owns, and seventeen named money amounts. The price table is loaded
from the mode's own cost section, which must exist. Starting money is the team's configured
amount, and zero for a spectator.

**Notes** — the money amounts cover situations this mode never reaches — round draws, rivals
wiped out, minor wins. They are read anyway because the team record is shared with the modes
that do use them, so one team section can be written once and used by any mode.

The invulnerable-kill modifier defaults to **0.5** when the key is absent: a kill made under
protection pays half. It is the only money value with a non-zero default, which marks it as a
rule rather than a tuning knob.

Default item names are converted to price-table indices at load, so the runtime never looks an
item up by name. An unknown name converts to the table's "not found" marker truncated into the
index field, which silently becomes a valid-looking index; a rebuild should reject unknown
names at load.

The price table is loaded three times over — once in creation and once per team — with the same
section name spelled out as a literal in each place, and the source marks the literal as
something that should come from configuration. Harmless and worth collapsing.

### `OnPlayerSelectSkin` / `OnPlayerChangeSkin`

**Contract** — a skin request is applied, acknowledged to the requesting client, and broadcast.
Applying it clears the player's spectator flag and **kills him**, so the new model takes effect
at once. A request for skin "any" picks uniformly from the team's skin list.

**Invariants** — changing skin costs a life. The visual is baked into the spawn record, so
there is no way to change it on a live body.

**Notes** — deathmatch has the same routine with a guard that makes its random-skin path
unreachable and its kill never fire. Here both work. So the two modes behave differently on a
skin change for no stated reason, and this one is what the code was evidently meant to do.

## Connection

### `OnPlayerConnect` / `OnPlayerConnectFinished` / `OnPlayerDisconnect`

**Contract** — a genuinely new connection has its record cleared, its team set to spectators and
its skin marked unselected; a reconnection keeps its record. Either way the player arrives
flagged as a spectator with the skip flag cleared, and the server's own client on a dedicated
server — or in spectator mode — is marked skip and takes no further part. Finishing the
connection broadcasts the arrival, zeroes the team-kill count, grants default items and (for a
new connection) starting money, spawns a spectator body and marks the client network-ready.
Disconnecting drops any carried artefact and removes the invincibility entry.

**Invariants** — a **reconnecting** player keeps his money, frags and rank, which is what makes a
dropped connection survivable.

**A disconnecting carrier's artefact is dropped, not destroyed.** Without it the objective would
leave the match with the player and the recovery would have to wait for the returning timer to
notice.

**Notes** — the team-kill count is reset on *every* connection completion including a
reconnection, so a player who disconnects to escape a team-kill punishment has his count wiped.
Everything else about him survives. That is an obvious hole and a rebuild should carry the
count on the account, as the ban list already does.

New players land on the spectators' team rather than being auto-assigned, so team selection is
an explicit act. The mode has no readiness stall for them because a spectator always counts as
ready.

## Anomalies

### `LoadAnomalySet` / `LoadAnomaliesItems` / `GetMinUsedAnomalyID`

**Contract** — the level's configuration names up to twenty sets plus a permanent list, each a
comma-separated list of zone **names**. Each name is resolved to one spawned zone entity by
picking the **least-used** zone of that name and incrementing its use count. A set that
resolves to nothing is discarded.

```text
FUNCTION load_anomaly_set()
  anomaly_permanent.clear() ; anomaly_sets.clear()
  FOR i IN 0 .. 19
    IF the level names no "set<i>": CONTINUE
    appended = load_anomalies_items("set<i>")
    IF it resolved to nothing: drop the appended set
  load_anomalies_items("permanent")

FUNCTION min_used_anomaly_id(zone_name) -> int
  candidates = every spawned zone registered under that name
  IF none: RETURN 0
  pick the candidate with the smallest use count
  that candidate's use count = use count + 1
  RETURN its entity identifier
```

**Invariants** — **a set names zone types, not zone instances.** A level with six zones called
the same thing and three sets naming it once each gets three different zones, one per set,
because the use counter spreads the picks. Without it every set would resolve to the same zone
and the rotation would change nothing.

The sets live in the **level's** configuration rather than the game's, because which zones exist
is a property of the map, and the section name is this mode's own — so a level ships a
different hazard layout per mode.

**Notes** — the twenty-set scan bound has no reason in the source, and unlike deathmatch's
version this one *continues* past a gap rather than stopping at it. That is the better
behaviour and a rebuild should adopt it in both places.

The warning emitted when a named set is missing says "permanent string not found" whatever was
actually looked for, which is a copy of the permanent list's own diagnostic. Cosmetic.

### `ReStartRandomAnomaly` / `StopPreviousAnomalies` / `SendAnomalyStates` / `CheckAnomalyUpdate`

**Contract** — rotation. A set that is not currently live is drawn at random, every set is
switched off, the drawn one is switched on if anomalies are enabled, and all of it is
broadcast as one packed message. The periodic check rotates once the configured set length has
elapsed.

```text
FUNCTION restart_random_anomaly()
  IF there are no sets: RETURN
  started = the indices of sets currently live
  REPEAT
    candidate = a uniform draw over the sets
  UNTIL candidate is not in `started`                # see Notes
  stop every set
  IF anomalies are enabled: mark candidate live
  send_anomaly_states()

FUNCTION send_anomaly_states()
  message = a packed event message
  append every PERMANENT zone set to IDLE
  append every zone of every NON-LIVE set set to DISABLED
  append every zone of every LIVE set set to IDLE        # this order is required
  last_anomaly_start = server_clock ; broadcast
```

**Invariants** — **the enable pass must come last.** A zone can appear in more than one set, and
if the disable pass ran after the enable pass it would switch off a zone the live set had just
switched on. The source marks the ordering as necessary and it is.

The permanent zones are set to idle on every rotation, not only once, so a zone that is both
permanent and a member of a disabled set ends up disabled — the later pass wins. That is
probably not intended; a rebuild should exclude permanent zones from the set passes.

**Notes** — **the draw loops forever when every set is already live.** With exactly one set
configured, that set is live after the first rotation and the loop can never find a different
one. A level shipping a single anomaly set therefore hangs the server on its second rotation.
A rebuild must handle "there is nothing else to pick" explicitly; the obvious answer is to
leave the current set running.

The packed form is a frozen wire layout: a message header followed by repeated (one-byte
length, event bytes) pairs. The one-byte length caps an inner event at 255 bytes, which a zone
state change is comfortably under, and the composer asserts the whole message stays under the
packet limit.

The rotation period is deathmatch's variable, read in minutes.

### `OnPostCreate`

**Contract** — every spawned zone is registered under its name with a use count of zero, so that
set loading can resolve names to entities.

**Invariants** — registration happens as zones spawn and the table is cleared at round start,
which is why round start must clear it *before* the world is rebuilt. Getting that order wrong
leaves the table naming destroyed entities.

**Notes** — this registers **every** zone, including ones owned by an artefact or a script.
Deathmatch's equivalent skips owned zones, on the grounds that they belong to something else.
Here an owned zone can be drawn into a rotation set and switched off under its owner. A rebuild
should adopt deathmatch's filter.

Unlike deathmatch, zones are **not** switched off as they spawn; they spawn in whatever state
the level gave them and the first rotation sorts it out. So the level's own anomaly state is
visible for the first fraction of a second of every round.

## Snapshot and update

### `net_Export_State`

**Contract** — appends the mode's join-time state after the multiplayer base's. Frozen field for
field against the decoder in
[`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md).

```text
green_artefact_id   : int (16-bit)    # both zero unless BOTH exist — see Invariants
blue_artefact_id    : int (16-bit)
green_home_point    : vector3
blue_home_point     : vector3
score_limit         : int (32-bit, signed)
green_score         : int (32-bit, signed)
blue_score          : int (32-bit, signed)
friendly_indicators : byte
friendly_names      : byte
bearer_MAY_sprint   : byte            # see Notes — the polarity is not what it looks like
artefact_can_be_activated : byte      # activated_artefact_ret is non-zero
base_radius         : real
in_warmup           : byte
time_limit          : int (16-bit, signed)   # MINUTES
```

**Invariants** — the two artefact identifiers are written as a pair and **both are zeroed if
either is missing**, so the client never sees one artefact without the other. It relies on
that: it treats "both exist" as meaning "neither is being carried".

The time limit travels as **minutes in a signed sixteen-bit field**, which caps it at about
nine hours of playable time and is frozen by the decoder.

**Notes** — **the sprint flag is sent as "the bearer may sprint", the negation of the setting.**
Artefact hunt sends the same conceptual flag with the opposite polarity — "the bearer may
*not* sprint" — over its own snapshot. The client stores both in identically named fields, so
the field's name is right in one mode and wrong in the other, and a reader comparing the two
client decoders will conclude one of them is inverted. It is the *servers* that disagree, not
the clients. A rebuild must pick one polarity and name it for what it carries.

Two policy flags the client needs at join time — friendly indicators and friendly names — are
sent only in this full snapshot and never again, so a policy changed mid-round does not reach
anyone already connected.

### `net_Export_Update`

**Contract** — the frequent increment: the time remaining to the next wave, the wave interval,
and the warm-up deadline.

```text
time_to_next_wave   : int (32-bit)   # MILLISECONDS; zero when the deadline has passed
wave_interval       : int (32-bit)   # MILLISECONDS
warmup_deadline     : int (32-bit)   # server clock; zero when not warming up
```

**Invariants** — the remaining time is **clamped to zero** rather than allowed to wrap, which is
the bug artefact hunt has in the equivalent field and this mode does not. A rebuild should
clamp; an unclamped unsigned subtraction past its deadline displays as a countdown of many
days.

**Notes** — both are in milliseconds here, and artefact hunt sends the interval in seconds and
the remainder in milliseconds in adjacent fields. One mode got it right. The warm-up flag
itself travels in the full snapshot rather than here, so a client that joins during warm-up
learns of it once and then watches only the deadline.

## Items

### `OnTouchItem`

**Contract** — the non-artefact pickup rule, reached when the touched thing is not an artefact.
A dropped player bag is **emptied into the toucher and destroyed**, paying a looting bonus if
the PDA-hunt setting is on. Everything else is allowed.

```text
FUNCTION on_touch_item(actor, item) -> allowed
  IF the item is a player bag lying on the ground
    FOR EACH child of the bag
      IF on_touch(actor, child, not forced) is denied: reject it out of the bag
      ELSE transfer it from the bag to the actor
    send the transfers as one packed event
    destroy the bag
    IF pda_hunt: pay the looter the configured bonus
    RETURN denied                       # the bag itself is never carried
  RETURN allowed                        # see Notes
```

**Invariants** — the transfers are batched into a single packed event rather than sent
individually. With a full loadout that is a dozen ownership changes, and sending them
separately would let a client render the actor holding a partial inventory for a frame.

**Notes** — **anything not a bag is allowed.** Deathmatch refuses whatever it does not recognise,
on the grounds that a player should not be able to carry a level prop into the round; this
mode has no such filter, no slot-occupancy rule and no weapon-swap handling. A rebuild should
take deathmatch's version, which is strictly more careful.

The bag bonus is the PDA-hunt rule — looting a dead opponent pays, which is what makes walking
over to a body worthwhile.

### `OnDetach` / `OnDetachItem` / `FillDeathActorRejectItems`

**Contract** — detaching an artefact broadcasts the drop and clears the owner. Detaching a bag
fills it from the player's inventory: everything the mode sells goes in, except the outfit and
except the reject list, while the knife and the torch are destroyed outright. The reject list
holds whatever the player was actively holding at death — unless that was the knife, or an
**artefact**.

```text
FUNCTION fill_death_reject_items(actor, out)
  slot = the actor's active slot
  IF slot is the knife slot: RETURN            # the knife is destroyed, not dropped
  IF no active slot: RETURN
  item = the item in that slot ; RETURN IF absent on either side
  IF the item is an artefact: RETURN           # see Invariants
  out.append(item)
```

**Invariants** — **the weapon in the dying player's hands does not go into the bag.** It is
rejected instead, which drops it where he fell. That separation is the visible rule: the weapon
you were using lands next to your body, the rest of your kit stays in the bag.

**An artefact in the dying player's hands is exempt from that rule**, because the death path has
already dropped it deliberately, at the death position, through the artefact's own path. Letting
the reject list drop it a second time would emit a second drop message and a second returning
timer.

**Notes** — the knife and the torch are destroyed rather than dropped because every player gets
them free, so dropping them would litter the level with items nobody needs. The outfit is
excluded from the other direction — it is worn, not carried.

### `DropArtefact`

**Contract** — releases an artefact from its carrier by processing an ownership-reject event
locally on the server. An optional position places it exactly, which is how a delivery puts the
scored artefact back on its home point.

**Notes** — the event is **processed rather than broadcast**: the server runs it through the same
path an event arriving from a client would take, so every downstream handler — including this
mode's own detach handler — sees it. That is why a delivery's drop reaches the drop broadcast
at all, and why the position override exists rather than a separate teleport.

## The server's own spectator mode

### `SM_CheckViewSwitching` / `SM_SwitchOnNextActivePlayer` / `SM_SwitchOnPlayer`

**Contract** — a non-dedicated server can run without playing, its window following live players
and switching between them on a timer or when the current target stops being an actor. The
switch picks a random connected, network-ready, non-skipped, living player; with none available
it falls back to the server's own body. Switching moves the view, turns the previous target's
first-person item display off and the new target's on, and arms the next switch. The screen is
captioned with the followed player's name.

**Invariants** — the candidate must be alive **and holding something**; a candidate who is not is
abandoned and the switch simply does not happen, to be retried on the next update. The point of
the mode is watching someone fight.

**Notes** — turning the first-person item display on for the watched player is what makes the
view look like that player's own screen rather than a camera at his eyes. It must be turned off
on the way out or two players' weapons render at once.

The whole feature is refused on a dedicated server, which has no window to watch through, and
its switch interval is floored at one second.

## `CheckForWarmap`

**Contract** — when a warm-up deadline is armed and has passed, clears it and issues a fast
restart.

**Notes** — the restart goes out **as a console command**, not as a method call, so warm-up
termination is indistinguishable from an operator typing it. Deathmatch does the same. A
rebuild should call the restart directly; nothing depends on the indirection.

## `ReadOptions`

**Contract** — parses the session option string, each key defaulting to the setting's current
value so an omitted key leaves it alone. The keys are short because they travel in the
server-browser advertisement, and they are **the union of three other modes' key sets** plus
this mode's own three.

**Invariants** — the reinforcement interval is **clamped to at least one**, which is what makes
artefact hunt's two sentinels unreachable in this mode (see
[`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md)).

**Notes** — the invincibility duration is read from the key named `dmgblock`, which in deathmatch
names the respawn damage-block time. Same key, same meaning, two independent variables — so a
server configured once behaves consistently across modes by coincidence rather than by design.

The spectator option both enables the server's own spectator mode and supplies its switch
interval in seconds, with minus one meaning absent.

## `SpawnWeaponsForActor`

**Contract** — spawns every item in the player's list onto his body, draining the list, then
charges the recorded purchase amount.

**Invariants** — the item encoding is **a packed pair of bytes in one sixteen-bit value**: the low
byte indexes the mode's price table, the high byte carries the weapon's addon flags. That
packing is the loadout representation everywhere in the session, so a rebuild must either keep
it or change every user together.

The list is drained, so one purchase cannot be spawned twice. The spawn call is handed the list
as well, because an item that implies others — a weapon that brings its ammunition — appends
them.

**Notes** — charging *after* spawning is what lets a dead player revise his purchase repeatedly
without being charged each time. It also means a purchase that fails to spawn is still charged.

Unlike deathmatch, there is no free-ammunition exclusion flag consulted here; that lives in the
companion file's answer to the free-ammunition question.

## `WriteGameState`

**Contract** — appends this mode's fields to the round statistics record: both team scores, the
time limit in minutes, the delivery limit, and whether anomalies were enabled.

## Pure delegation and dead surface

**Contract** — several declared units add nothing.

- `OnPreCreate` / `OnCreate` — pure delegation to the base. They exist so that a future rule
  has a place to go; nothing in this mode vetoes a spawn or notices a creation.
- `OnDestroyObject` — re-targets the spectator camera if its subject disappeared, then
  delegates and tells the item respawner.
- `ReturnArtefactToBase` — declared, never defined, never called.
- `ProcessPlayerKill` — commented out entirely. It would have counted a raw kill on the killer,
  which the kill-result path already does.
- The commented-out block in the connection path would have given a connecting player money
  and default items; the same work is done a moment later when the connection finishes, so the
  disabled version was redundant rather than wrong.
