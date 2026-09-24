# src/xrGame/game_sv_teamdeathmatch.cpp

> Team deathmatch: two teams whose scores are the sum of their members' frags, with friendly fire, team-kill punishment, automatic balancing and swapping, and a drop-bag that replaces ordinary item pickup.

**Needs** — [`game_sv_teamdeathmatch.h`](game_sv_teamdeathmatch.h.md) · [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`clsid_game.h`](../xrServerEntities/clsid_game.h.md) · [`Level.h`](Level.h.md) · [`ui/UIBuyWndShared.h`](ui/UIBuyWndShared.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`game_sv_teamdeathmatch.h`](game_sv_teamdeathmatch.h.md); callers name that, not this file.
**Tier floor** — T2: session rules over the network and entity layers

## Purpose

Deathmatch already knows how to run rounds, count frags, sell weapons and end on a limit.
Team deathmatch changes four things and everything else is inherited:

1. **A frag belongs to a team.** The team score is not counted independently — it is
   maintained *incrementally* from each player's frag delta, which is what lets a negative
   frag adjustment (a suicide, a team kill) pull the team score down without re-summing.
2. **A kill may be a team kill**, which pays the team-kill money amount instead of the rival
   amount, does not advance the killer's frags, and can get the killer kicked.
3. **Damage between team-mates is scaled** by a server factor rather than refused, so the
   operator can choose between no friendly fire, reduced, full, or amplified.
4. **Item ownership goes through a drop bag.** Touching another player's dropped bag empties
   it into the toucher rather than picking the bag up; dying fills a bag from the corpse.

## State

```text
RECORD TeamDeathmatchSession
  teams        : list<TeamScore>    # exactly two, appended at creation; index = team - 1
  teams_swaped : bool               # have the sides been exchanged on this map already

# server settings, all reachable as console variables and as session options
  auto_team_balance    : bool = false
  auto_team_swap       : bool = true
  friendly_indicators  : bool = false   # draw a marker over team-mates
  friendly_names       : bool = false   # draw team-mates' names
  friendly_fire_factor : real = 1.0     # multiplier on damage to a team-mate
  team_kill_limit      : int  = 3       # team kills before punishment
  team_kill_punishment : bool = true
```

**Invariants** — team numbers are **one-based** on a player (`0` means no team yet, `1` and
`2` are the playing teams) and **zero-based** as an index into the score list. Every
conversion in this file is `team - 1`, and getting it wrong silently scores the wrong side.

The team-kill limit defaults to three and the punishment defaults to on: that pair is a
shipped-balance decision, not a technical one.

## `Create`

**Contract** — runs the deathmatch creation, asserts that the level has at least one
spectator respawn point, then appends **two** zeroed team score records and enters the
pending phase.

**Invariants** — the spectator respawn set must be non-empty because every player joins as a
spectator before selecting a team; a level without one cannot run this mode at all, so the
failure is immediate and loud rather than at the first connection.

The round-end reason is initialized to "forced" so that the very first round start does not
trigger the balance-and-swap pass — there is nothing to balance yet.

## `LoadTeams`

**Contract** — points the weapon price table at this mode's cost section and loads **three**
team configuration sections, named for teams zero, one and two.

**Invariants** — three sections for two playing teams. Section zero is the spectator/unassigned
team: a player with team `0` still needs a default item list and a money floor, because they
exist in the session before choosing a side. A rebuild that loads only two will fault the
moment a spectator is asked for their team data.

A missing cost section is fatal: the buy menu cannot price anything without it.

## `ReadOptions`

**Contract** — reads the five team settings out of the session option string, each defaulting
to its current value, so an option string that omits a key leaves the setting alone rather
than resetting it. The keys are short (`abalance`, `aswap`, `fi`, `fn`, `ffire`) and are part
of the server-browser protocol.

## `net_Export_State`

**Contract** — appends two bytes to the base session's join-time state: whether friendly
indicators and friendly names are enabled. These are pure presentation and are sent because
the client draws them and must not have to ask.

## Team membership

### `AutoTeam`

**Contract** — picks the team a joining player should be put on: counts ready, non-spectating
players on each playing team and returns the smaller one, **breaking ties toward team one**.

```text
FUNCTION auto_team() -> int
  counts = [0, 0]
  FOR EACH connected client
    IF client has no player state THEN CONTINUE
    IF client is not network-ready THEN CONTINUE
    IF player is skipped OR player.team = 0 OR player is a spectator THEN CONTINUE
    counts[player.team - 1] += 1
  IF counts[0] > counts[1] THEN RETURN 2 ELSE RETURN 1
```

**Notes** — the comparison is strict, so equal teams and an empty server both send the player
to team one. That is why the first two players on an empty server end up on opposite sides.

### `GetPlayersCountInTeams`

**Contract** — nominally, how many players are on a given team.

**Notes** — **this implementation does not do that**, and the discrepancy is worth recording
rather than reproducing. It skips clients that *are* network-ready (the opposite of every
other pass in the file), and it counts players whose team number is greater than *or equal
to* the requested one rather than equal. The only caller asks whether the two teams are the
same size, where the second error partly cancels; the first means the answer is computed over
clients that are still connecting. A rebuild should write the obvious version — count ready,
non-spectating players whose team equals the argument — and accept that team-size equality
will then be decided slightly differently than in the original.

### `TeamSizeEqual`

**Contract** — whether the two playing teams have the same population, via the count above.

### `AutoBalanceTeams`

**Contract** — when enabled, moves players from the larger team to the smaller until the
difference is halved. Runs between rounds only.

```text
FUNCTION auto_balance()
  IF NOT auto_team_balance THEN RETURN
  counts = per-team population over ready, non-skipped players
  IF counts equal THEN RETURN
  larger, smaller = the two teams by population
  to_move = (counts[larger] - counts[smaller]) / 2     # integer division: halving the
                                                       # gap makes the teams equal, since
                                                       # each move changes it by two
  WHILE to_move > 0
    victim = the player on `larger` with the FEWEST frags
    victim.team = smaller
    to_move -= 1
```

**Invariants** — moving the lowest-scoring player is the whole policy: it takes the least
from the team being shrunk and is the least disruptive to a player who is not invested in the
round. The halving is exact because each move shifts the difference by two.

**Notes** — the loop re-scans for the lowest scorer on every iteration rather than sorting,
and the scan reads the players' teams as it mutates them, so the second move sees the first
one's effect. That is correct here and must be preserved: moving the same player twice, or
moving a player who is already on the smaller team, would be wrong.

### `AutoSwapTeams`

**Contract** — when enabled, exchanges every playing player's team (one becomes two and two
becomes one) and records that the swap has happened.

**Invariants** — players with team zero are left alone; spectators do not have a side to swap.

**Notes** — the recorded flag interacts with map rotation: a map due to rotate is held back
for one more round if a swap has not yet happened, so that both teams get to play both sides
before the map changes. That is the rule in `OnRoundEnd`, and the flag is cleared only on a
full game restart.

### `OnPlayerConnect`

**Contract** — after the base handles the connection, assigns the new player a team by the
auto rule, grants starting money **unless this is a reconnect** (a reconnecting player keeps
the balance they had), and gives them their team's default items.

### `OnPlayerConnectFinished`

**Contract** — the ordered sequence that makes a fully-joined player visible to everybody.
The order is load-bearing:

```text
FUNCTION on_connect_finished(client)
  player.team        = 1            # a provisional team, overwritten the moment the
                                    # player picks one; it exists so the player record
                                    # is never broadcast with an invalid team
  player.team_kills  = 0
  player.set(spectator)
  player.set(ready)

  broadcast player_connected + the player's full exported record
  broadcast player_join_team with the player's name and team

  spawn the player as a spectator      # a body must exist before the client can render
  send the current anomaly states      # the level's anomalies are already running
  client.network_ready = true          # only now may the player be counted or targeted
```

**Invariants** — the network-ready flag is set **last**. Every population count, balance pass
and respawn-point search in this file skips clients that are not ready, so setting it early
would let a half-joined player be counted, balanced or killed.

The two broadcasts are separate messages rather than one because the team-join message is
also what the chat log prints; merging them loses the log line.

### `OnPlayerSelectTeam` and `OnPlayerChangeTeam`

**Contract** — a player has asked for a team. A request for team zero means "whatever you
think", and is resolved by a rule that avoids pointless churn:

```text
FUNCTION change_team(client, requested)
  IF requested = 0 THEN
    IF player has no team yet THEN requested = auto_team()
    ELSE IF the teams are equal in size THEN requested = player.team   # stay put
    ELSE requested = auto_team()

  reply to this client alone with the team it is getting     # always, even if unchanged:
                                                             # the client's menu is waiting
  IF requested = player.team THEN RETURN                     # nothing else to do

  kill the player                       # changing sides costs a life, always
  player.set(spectator)
  old_team    = player.team
  player.team = requested

  team_data = configuration for the new team
  IF team_data exists AND (player.money < team_data.start OR old_team = 0) THEN
    grant starting money                # switching to a richer team tops you up; it never
                                        # takes money away
  broadcast the team change
  grant the new team's default items
```

**Invariants** — the private reply is sent before the early return, so a player who asked for
the team they already have still gets their menu dismissed. The kill is unconditional on an
actual change: a player may not carry a body, a position or a loaded weapon across the line.

The money rule is one-directional by design — it tops a player up to the new team's starting
amount but never confiscates the excess, so switching sides cannot be used to reset a bad
economy but also cannot be punished by an asymmetric one.

## Scoring

### `GetKillResult`

**Contract** — takes the deathmatch classification and **reclassifies a rival kill as a
team-mate kill when both players share a team**. Everything else passes through.

### `OnKillResult`

**Contract** — applies a classification. For a team-mate kill: increments the killer's
team-kill counter, pays the team's team-kill amount (normally negative), and answers *false*,
which tells the base not to award the kill. Every other result is handled by deathmatch.

**Invariants** — the false answer is what suppresses the frag. The killer's frag count is
therefore unchanged by a team kill, while the team score still moves if the base applied a
death penalty to the victim.

### `UpdateTeamScore`

**Contract** — adds a player's frag delta to their team's score.

```text
FUNCTION update_team_score(player, frags_before)
  team_score[player.team - 1] += player.frags - frags_before
```

**Invariants** — incremental, not recomputed. This is the only correct way to keep the team
score consistent with a frag count that the base may adjust by any amount in either
direction, including for reasons this mode does not know about.

### `OnPlayerKillPlayer`

**Contract** — samples both players' frag counts *before* delegating to the base, then rolls
each delta into the respective team score, then applies team-kill punishment.

```text
FUNCTION on_kill(killer, victim, kill_type, special, weapon)
  killer_frags_before = killer.frags        # sampled BEFORE: the base changes both
  victim_frags_before = victim.frags
  base.on_kill(...)
  update_team_score(killer, killer_frags_before)
  IF killer is not victim THEN
    update_team_score(victim, victim_frags_before)   # a suicide would double-count

  IF killer and victim exist AND are different AND share a team THEN
    IF team_kill_punishment AND killer.team_kills >= team_kill_limit THEN
      disconnect the killer with a translated "kicked by server" reason
```

**Invariants** — the before-samples must be taken before the base runs, and the victim's
delta is skipped for a suicide because killer and victim are the same record and the delta
would be applied twice.

**Notes** — the kick searches the client list for the client owning the killer's record, and
explicitly **excludes the server's own client**. On a listen server the host cannot be kicked
for team-killing; kicking them would end the session for everybody.

The team-kill counter is incremented in the classification handler, not here, so by the time
this test runs the current kill is already counted — the limit is "the third team kill gets
you kicked", not the fourth.

### `checkForFragLimit`

**Contract** — the round ends on the frag limit when **either team's** score reaches it — not
when a player does. A zero limit disables the check.

### `HasChampion`

**Contract** — there is a winner when the two team scores differ, or when the server is
configured to skip waiting for a decisive result.

### `OnFraglimitExceed` and `OnTimelimitExceed`

**Contract** — identical bodies with different end reasons: the team with the *higher* score
wins, the session moves to that team's scoring phase, and the round end is scheduled rather
than immediate.

```text
FUNCTION on_limit_exceeded(reason)
  winner = (team_score[0] < team_score[1]) ? 1 : 0    # ties resolve to team one
  announce the team score for `winner`
  switch_phase(winner = 0 ? team1_scores : team2_scores)
  schedule_round_end(reason)
```

**Notes** — a tie sends the win to team one. That is consistent with the tie-break in
`AutoTeam` but is nowhere stated as a rule; a draw phase exists in the phase enumeration and
is never entered from here.

### `Update`

**Contract** — while the session is in one of the three scoring phases, ends the round once
the scheduled delay has elapsed. The delay is what gives the clients time to show the
end-of-round board.

## Rounds

### `OnRoundStart`

**Contract** — balances and swaps the teams *unless* the previous round ended by force or by
a fast restart, then runs the base round start. A full game restart also clears the
swapped flag.

**Invariants** — the exclusion matters: an operator forcing a restart wants the same teams
back, not a reshuffle.

### `OnRoundEnd`

**Contract** — after the base round end, cancels a pending map rotation if auto-swap is on
and the teams have not yet been swapped on this map — holding the map for one more round so
both teams play both sides.

### `RespawnPlayer`

**Contract** — after the base respawn, clears the on-base flag. A respawning player is not in
their base even if they died there.

### `RP_2_Use`

**Contract** — which respawn-point set a spawning entity should use: the set indexed by the
entity's team, falling back to set zero when that team has no points authored. Non-actors
always get set zero.

**Invariants** — the fallback is what lets a level author ship a single shared respawn set
and still have the mode run.

## Damage

### `OnPlayerHitPlayer_Case`

**Contract** — scales a hit between two different players on the same team, before the base
applies it.

```text
FUNCTION scale_friendly_hit(hitter, victim, hit)
  IF hit.type = physical_strike THEN skip scaling   # melee is never scaled
  ELSE IF hitter and victim are different AND share a team THEN
    hit.power   *= friendly_fire_factor
    hit.impulse *= max(friendly_fire_factor, 1.0)
  base.on_hit(hitter, victim, hit)
```

**Invariants** — power and impulse are scaled **differently**. A factor below one reduces the
damage but leaves the impulse at full strength, so a team-mate's shot still visibly shoves
you even when it barely hurts — the feedback that tells you someone is shooting you survives
the setting. A factor above one scales both.

Melee is exempt entirely, so a knife always does full damage to a team-mate regardless of
the setting.

**Notes** — friendly fire is reported as *enabled* when the factor rounds to at least one
hundredth, and the factor is reported as zero below that. The hundredth threshold exists
because the setting is transported and displayed as a percentage.

## Items — the drop bag

### `OnTouch` / `OnTouchItem`

**Contract** — a player has touched something. Only two outcomes exist in this mode:

```text
FUNCTION on_touch(toucher_id, item_id) -> allowed
  toucher = server object, must be a multiplayer actor
  item    = server object
  IF either is missing or the toucher is not an actor THEN RETURN denied

  IF item is a player drop bag AND the bag has no parent THEN
    FOR EACH child of the bag
      IF on_touch(toucher, child) is denied THEN
        emit an ownership-reject event for that child and CONTINUE
      perform the transfer child: bag -> toucher
      append both halves of the transfer to one batched event message
    send the batched message IF it contains anything
    destroy the bag
    IF pda-hunt is enabled THEN pay the toucher the configured pda bonus
    RETURN denied                 # the bag itself is never picked up
  RETURN allowed
```

**Invariants** — the bag is a container, not an item: touching it empties it and destroys it,
and the answer is always *denied* so that the ownership machinery does not also attach the bag
to the player. A rebuild that answers "allowed" here will leave the player carrying a
destroyed object.

The per-child transfers are packed into **one** batched message rather than sent
individually, because a full bag is a dozen transfers and each is two messages; sending them
separately lets a client render a partially-emptied bag.

The batched message is suppressed when it holds only its header, which is why the emptiness
test is against a small byte count rather than zero.

### `OnDetach` / `OnDetachItem`

**Contract** — a player is giving up their inventory into a bag, which is what happens when
they die. Each carried item is sorted into one of three outcomes and the outcomes are applied
in a fixed order.

```text
FUNCTION on_detach(actor, bag)
  IF bag is not a player drop bag THEN RETURN

  to_reject = items the death rules say must be returned rather than dropped
  to_destroy = empty ; to_transfer = empty

  FOR EACH item carried by the actor
    IF item is in to_reject THEN CONTINUE
    IF item is a knife OR a torch THEN to_destroy += item
    ELSE IF the item appears in the mode's weapon price table THEN
      IF item is not an outfit THEN to_transfer += item

  batch every transfer actor -> bag into one message and send it
  destroy everything in to_destroy
  reject everything in to_reject
```

**Invariants** — the price table is the filter for what may be dropped. An item nobody can buy
is not droppable, which keeps mode-specific and quest items out of the economy. Outfits are
excluded on top of that: a dropped outfit would let a team accumulate armour across rounds.

The knife and the torch are destroyed rather than dropped because every player spawns with
them; dropping them would litter the level with duplicates.

The transfers go out before the destructions, so that the bag's contents are correct on the
clients before anything referenced disappears.

**Notes** — the rejection list is computed by the shared death-handling helper, and the source
itself notes that the call may belong in the death path rather than here. It is invoked here
because this is the only place that knows the bag exists.

## Team base

### `OnObjectEnterTeamBase` / `OnObjectLeaveTeamBase`

**Contract** — an actor has crossed a base volume's boundary. The flag is set **only when the
actor's team matches the volume's team**: standing in the enemy base is not "on base".

```text
FUNCTION on_enter_base(entity_id, zone_team)
  actor = server object; RETURN if not an actor
  team = client_game.modify_team(zone_team)   # map the authored zone numbering onto
                                              # the session's team numbering
  IF actor.player.team = team THEN
    actor.player.set(on_base)
    request a state synchronisation to the clients
```

**Invariants** — the zone's team number passes through the *client* game's team mapping.
That is the same mapping the client uses to decide which side a zone belongs to, and the two
must agree or a player will be on-base on one side of the wire and not the other.

The synchronisation request is explicit because the flag is read by round-scoring rules on
other clients and must not wait for the next periodic update.

## `WriteGameState`

**Contract** — appends each team's score to the round-result record the server writes for
external consumption, keyed by team index.

## `ConsoleCommands_Create`

**Contract** — deliberately empty. This mode registers no console commands of its own; its
settings are reached through the generic server-setting mechanism.

**Notes** — the clear counterpart still delegates to the base, which is asymmetric but
harmless: there is nothing of this mode's to clear.
