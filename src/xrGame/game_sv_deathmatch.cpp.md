# src/xrGame/game_sv_deathmatch.cpp

> The authoritative side of free-for-all deathmatch: the round state machine, the money-and-buy economy, kill accounting and bonuses, spawn-point choice, respawn protection, and the rotating anomaly sets that make a multiplayer level move.

**Needs** — [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md) · [`game_sv_mp.h`](game_sv_mp.h.md) · [`game_sv_mp_team.h`](game_sv_mp_team.h.md) · [`game_base.h`](game_base.h.md) · [`game_base_kill_type.h`](game_base_kill_type.h.md) · [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrServerEntities/clsid_game.h`](../xrServerEntities/clsid_game.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`HudItem.h`](HudItem.h.md) · [`Missile.h`](Missile.h.md) · [`eatable_item_object.h`](eatable_item_object.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`UIGameDM.h`](UIGameDM.h.md) · [`ui/UIBuyWndShared.h`](ui/UIBuyWndShared.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: session rules over the network and entity layers; no device contact and no frozen memory image beyond one struct blit

## Purpose

This is the base of every competitive multiplayer mode the engine ships. Team deathmatch,
artefact hunt and capture-the-artefact are all narrowings of it, so almost every rule here is
written to be overridden: what counts as a kill, which respawn set to draw from, how a round
ends, who gets paid.

Six concerns live in this file and they are only loosely related to each other:

1. **The round state machine** — pending, in progress, player-scores — and the conditions
   that move between them.
2. **The economy** — every player carries money for the round, earns it by killing and
   spends it in a buy menu, and the server is the only thing that may adjust it.
3. **Kill accounting** — classifying a kill, paying for it, and awarding situational
   bonuses for headshots, knife kills and kill streaks.
4. **Spawn placement** — choosing a respawn point far from living enemies, and the
   short invulnerability that follows a respawn.
5. **Anomaly rotation** — the level ships several named sets of zones; one set is live at a
   time and they rotate on a timer, so a map plays differently across rounds.
6. **Item flow** — what a player may pick up, and what happens to their inventory when they
   die.

The server is authoritative for all of it. The client mirrors it from the session snapshot
([`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md)) and is never trusted with money,
frags or item ownership.

## State

```text
RECORD DeathmatchSession
  teams            : list<TeamScore>      # EMPTY in plain deathmatch; team modes fill it
  free_rpoints     : list<int> per team   # respawn indices not yet used this cycle
  last_rpoint      : int per team         # last point used, excluded from the next refill
  base_cost_section: text                 # names the weapon price table

  warmup_deadline  : int (ms, server clock)   # 0 when no warm-up is pending
  in_warmup        : bool

  round_end_delayed : bool                # a round end has been scheduled
  round_end_at      : int (ms)
  team_wipe_delayed : bool                # set here, consumed only by artefact hunt
  team_wipe_at      : int (ms)
  winning_name      : text                # published in the scores phase only

  anomaly_sets      : list<list<text>>    # authored names, per set, read from the level
  anomaly_ids       : list<list<int>>     # the entity identifiers those names resolved to
  anomaly_permanent : list<text>          # never switched off
  unused_set_ids    : list<int>           # the shuffle bag for set selection
  live_set_id       : int                 # 1001 = "none yet"; see Notes
  set_started_at    : int (ms)

  spectator_mode    : bool                # the server window follows players instead of playing
  spectator_target  : GameObject
  spectator_switch_at : int (ms)
  spectator_switch_delta : int (ms)
```

Alongside it a set of process-wide session settings, each readable as a console variable and
each overwritten from the session option string at creation:

```text
frag_limit        : int  = 10       # 0 disables
time_limit        : int  = 0        # minutes; 0 disables
force_respawn     : int  = 0        # seconds a corpse waits before being respawned; 0 disables
damage_block_time : int  = 0        # seconds of post-respawn invulnerability
damage_block_indicators : bool = true
anomalies_enabled : bool = true
anomaly_set_length: int  = 3        # minutes a set stays live
warmup_time       : int  = 0        # seconds
pda_hunt          : bool = true     # pay a bonus for looting a dead player's bag
ignore_money_on_buy : bool = false  # debug: let players buy for free
```

Invariants:

- **`teams` is empty in plain deathmatch.** Nothing in this file ever adds to it; the team
  modes do. Every reader must therefore cope with a zero-length score list — the snapshot
  writes a count of zero, and spawn-point selection treats *everyone* as an enemy because
  the "same team" test is guarded on the list being non-empty. A rebuild that assumes at
  least one team will place players on top of each other.
- **The settings are process-wide, not per session.** They are file-scope values shared by
  every mode instance, which is sound only because one session runs at a time.
- **Money is only ever adjusted through the base session's add-money operation**, which
  clamps to the team's configured floor. Nothing here writes a player's balance directly
  except the start-of-round initialisation.

## The session settings

### `ReadOptions`

**Contract** — parses the session option string, each key defaulting to the setting's current
value so that an omitted key leaves it alone. The keys are short because they travel in the
server-browser advertisement: `frcrspwn`, `fraglimit`, `timelimit`, `dmgblock`, `dmbi`,
`ans`, `anslen`, `warmup`, `pdahunt`, `spectr`.

**Notes** — `spectr` is the odd one. It both enables the server's own spectator mode and
supplies its switch interval in seconds, with `-1` meaning "absent". The interval is floored
at one second, and the whole feature is refused on a dedicated server, which has no window to
watch through.

### `GetTimeLimit` / `GetFragLimit` / `GetForceRespawn` / `GetDMBLimit` / `GetWarmUpTime` / `IsAnomaliesEnabled` / `GetAnomaliesTime` / `IsDamageBlockIndEnabled`

**Contract** — one accessor per setting, each virtual so a derived mode can substitute its own
source. That indirection is the whole reason they exist: artefact hunt reads its own frag
limit, and the base's round-end checks call through here rather than reading the values.

## `Create`

**Contract** — runs the base session creation, asserts the level has at least one player
respawn point, loads the team configuration, reads the free-ammo exclusion list, enters the
pending phase, seeds the random generator from the processor tick counter, and loads the
level's anomaly sets.

**Invariants** — a level with no respawn points cannot run this mode and the failure is
immediate rather than at the first connection.

**Notes** — seeding from the tick counter is what makes spawn placement and anomaly selection
differ between server runs on the same map. A rebuild wanting reproducible matches needs a
seed it controls.

## The round state machine

### `Update`

**Contract** — the per-frame pass, dispatching on the current phase.

```text
FUNCTION update()
  base.update()
  IF phase is IN_PROGRESS
    check_for_warmup()          # may end warm-up and restart the round
    check_for_round_end()       # time limit, then frag limit
    check_invincible_players()  # expire respawn protection
    check_force_respawn()       # revive corpses that have waited long enough
    check_for_anomalies()       # rotate the live anomaly set
    IF spectator_mode: follow an active player and caption the view
  ELSE IF phase is PENDING
    check_statistics_ready()
    check_for_round_start()
  ELSE IF phase is PLAYER_SCORES
    IF round_end_delayed AND round_end_at has passed: end the round
```

**Notes** — the order inside the in-progress branch is load-bearing at one point only: the
round-end check runs before force-respawn, so a round that has just ended does not revive a
corpse into a finished match. Everything else in the list is independent.

The scores phase is a *dwell*: the round has already been decided, the winner's name is being
shown, and the actual round end fires on a timer. That is why the end is a separate delayed
step rather than immediate.

### `checkForRoundStart`

**Contract** — in the pending phase, starts the round once either every player is ready or a
fixed wait has elapsed since the phase began. If a map rotation is configured and has a next
map, switches to it instead of starting a round here.

```text
FUNCTION check_for_round_start() -> bool
  IF the level's game configuration has not finished loading: RETURN false
  IF NOT (fast_restart
          OR all_players_ready()
          OR server_time - phase_start_time > pending_wait_time)
    RETURN false
  IF a map rotation exists AND a next map was selected
    go to the next map
  ELSE
    start the round
  RETURN true
```

**Notes** — the pending wait is ten seconds. It is the answer to "a player joined, walked
away, and never pressed ready": the round starts anyway. A fast restart bypasses both
conditions, which is how the warm-up ends.

The load-guard is not defensive noise — a restart issued while the server is still bringing
the level up would reset state the loader is in the middle of writing.

### `AllPlayers_Ready`

**Contract** — true when every connected player counts as ready. A player counts if they have
signalled ready, are a spectator, are marked skip, or are not yet network-ready. The
server's own client is exempt from the not-yet-ready rule. False when there are no players.

**Notes** — counting a *not yet connected* player as ready is deliberate: a client still
loading the level must not hold the lobby, because it may never finish. The consequence is
that a round can start while someone is still loading, and they join mid-round.

### `OnRoundStart`

**Contract** — reloads the anomaly sets, clears the delayed-end flags and the winner's name,
zeroes every team score, arms warm-up if one is configured and this is not a fast restart,
runs the base start, starts an anomaly set, resets the respawn-point cycle for every team,
respawns every client as a spectator, and respawns every world item.

```text
FUNCTION on_round_start()
  reload anomaly sets ; clear delayed-end flags ; winning_name = ""
  live_set_id = 1001                        # "none"
  FOR EACH team: team.score = 0 ; team.num_targets = 0
  IF NOT fast_restart AND warmup_time > 0
    warmup_deadline = server_time + warmup_time seconds
    in_warmup = true
  base.on_round_start()
  IF anomalies_enabled: start_anomalies()
  FOR EACH team: free_rpoints.clear() ; last_rpoint = none
  FOR EACH client: respawn_as_spectator(client)
  respawn every world item
```

**Invariants** — a **fast restart never re-arms warm-up**. That is what makes warm-up
terminate: the warm-up deadline expiring issues a fast restart, which starts a round with no
warm-up. Without the exemption the server would warm up forever.

**Notes** — every player is put back into a spectator body at round start, not into the world.
Entry into play is a separate, explicit act (see `OnPlayerReady`), which is what gives the
client its pre-round buy and skin window.

### `OnRoundEnd`

**Contract** — if the round was in progress, moves every non-skipped player into a spectator
body before delegating to the base. From any other phase it delegates directly.

**Notes** — the phase guard matters: a round ending from the scores phase has already put
everyone in spectator bodies, and doing it twice would spawn a second spectator per player.

### `checkForRoundEnd` / `checkForTimeLimit` / `checkForFragLimit`

**Contract** — time limit first, frag limit second; both are suppressed entirely during
warm-up. The time limit additionally refuses to fire while the match is tied.

```text
FUNCTION check_for_round_end() -> bool
  IF in_warmup: RETURN false
  RETURN check_for_time_limit() OR check_for_frag_limit()

FUNCTION check_for_time_limit() -> bool
  IF in_warmup: RETURN false
  IF time_limit is 0: RETURN false
  IF server_time - round_start_time <= time_limit minutes: RETURN false
  IF NOT has_champion(): RETURN false        # a tie plays on
  on_time_limit_exceeded() ; RETURN true

FUNCTION check_for_frag_limit() -> bool
  IF frag_limit is 0: RETURN false
  IF any player's frags >= frag_limit
    on_frag_limit_exceeded() ; RETURN true
  RETURN false
```

**Notes** — **overtime is a real rule.** A deathmatch whose clock expires with two players
level does not end; it keeps running until one of them is alone at the top. That is the only
purpose of the champion test, and a rebuild that ends on the clock will produce ties the mode
has no way to resolve.

Nothing caps overtime, so a match between two evenly matched players can in principle run
indefinitely. The debug escape (`g_sv_Skip_Winner_Waiting`) forces the champion test to pass.

### `HasChampion`

**Contract** — true when exactly one player holds the highest frag count, or when the debug
override is set. Counts every client including spectators.

**Notes** — the frag scan seeds its maximum at -100, so a player below that is never
considered. Frags are a signed value that suicides push negative, and -100 is far enough
below any real score to be safe rather than a considered bound.

### `OnTimelimitExceed` / `OnFraglimitExceed` / `OnPlayerScores`

**Contract** — both limit handlers schedule a delayed round end with the matching reason and
then enter the scores phase. Entering the scores phase finds the leading player, publishes
their name for the snapshot, and switches phase.

**Notes** — the reason code (`time limit`, `frag limit`) is recorded so the statistics dump
and the client's end-of-round screen can distinguish them. The round then ends on the
delay, not on the limit.

If there is no winner at all — an empty server — the phase is not switched and the session
simply carries on.

### `OnDelayedRoundEnd` / `OnDelayedTeamEliminated`

**Contract** — dump the round statistics asynchronously, record the reason, and arm a timer
seven seconds out. The team-elimination variant arms a separate timer with the same delay.

**Notes** — seven seconds is the scoreboard dwell, and it is the only thing separating the
moment a round is decided from the moment the next lobby begins.

The team-elimination timer is **set here and never read here** — only artefact hunt consumes
it. A rebuild should put it with the mode that uses it.

### `check_for_WarmUp`

**Contract** — when a warm-up deadline is armed and has passed, clears it and issues a fast
restart.

**Notes** — the restart is issued **as a console command**, not as a method call. That is
worth naming because it means warm-up termination goes out through the command layer and is
therefore indistinguishable from an operator typing it. A rebuild should call the restart
directly; nothing depends on the indirection.

## Kill accounting

### `OnPlayerKillPlayer`

**Contract** — the entry point for every death. Processes the victim, forces a state
synchronisation, classifies the kill, applies its consequences, records it in the weapon
statistics, and — only if the classification allows — awards situational bonuses.

```text
FUNCTION on_player_kill_player(killer, victim, kill_type, special_kind, weapon)
  process_victim(victim, killer)
  signal_synchronize()                        # push the new state to clients now
  IF killer is none OR victim is none: RETURN
  result = classify_kill(killer, victim)
  may_bonus = apply_kill_result(result, killer, victim)
  weapon_statistics.record(killer, kill_type, special_kind)
  IF may_bonus: give_bonus(result, killer, victim, kill_type, special_kind, weapon)
```

**Invariants** — the victim is processed **before** the kill is classified, so the victim's
death count and cleared streak are already in place when the killer is paid. The
synchronisation is forced at that same point rather than waiting for the next tick, because a
death is the one event a client must not see late.

### `Processing_Victim` / `Victim_Exp`

**Contract** — marks the victim permanently dead, increments their death count, resets their
current kill streak, stamps the death time, refills their default item list ready for the
next life, awards their death experience, and records it in the statistics. A death with no
killer additionally counts as a self-kill.

**Notes** — awarding experience on *death* looks wrong and is not. The call adds zero
experience but re-enables rank promotion around it, so the effect is: a player who has
earned enough experience to rank up is promoted **at the moment they die**, never mid-fight.
That is the whole mechanism, and it exists so that a rank change — which alters the player's
available loadout — lands between lives instead of in the middle of one.

The default item list is refilled here rather than at respawn, so a player who opens the buy
menu while dead is editing a list that already has their rank's free items in it.

### `GetKillResult` / `OnKillResult`

**Contract** — classification and consequence. Plain deathmatch knows three outcomes: no
kill, a self-kill, and a rival kill. A self-kill increments the self-kill count and pays the
(negative) self-kill amount; a rival kill increments the rival kill count, advances the
streak, updates the streak record, and pays the rival amount. Only a rival kill earns
bonuses.

```text
FUNCTION apply_kill_result(result, killer, victim) -> may_bonus
  team = team data for the killer's team
  IF result is SELF
    killer.self_kills = killer.self_kills + 1
    add_money(killer, team.kill_self_amount)
    RETURN false
  IF result is RIVAL
    killer.rival_kills = killer.rival_kills + 1
    killer.streak = killer.streak + 1
    killer.best_streak = max(killer.streak, killer.best_streak)
    amount = team.kill_rival_amount
    IF killer is currently invulnerable
      amount = amount * team.invincible_kill_modifier
    add_money(killer, amount)
    RETURN true
  RETURN false
```

**Invariants** — a kill made while under respawn protection pays a **reduced** reward, halved
by default. That is the anti-spawn-camping rule: the protection is there so you can get out
of your spawn, not so you can trade shots for free. It is a money penalty only — the frag
still counts.

**Notes** — the frag counters are not touched here; a player's frag total is derived from the
rival and self kill counts. The lines that would have adjusted frags directly are commented
out, which is what makes the derived form authoritative.

A kill with no team data (a player on a team the configuration does not describe) silently
pays nothing rather than failing. Given that plain deathmatch loads exactly one team section,
this is the normal path for a misconfigured level.

### `OnGiveBonus`

**Contract** — pays situational bonuses on top of a rival kill: experience and money for a
headshot or an eye shot, money for a backstab, money for a knife kill, and a money bonus
named for the killer's current streak length. Every amount is read from configuration at the
moment it is paid, defaulting to zero.

```text
FUNCTION give_bonus(result, killer, victim, kill_type, special, weapon)
  IF result is not RIVAL: RETURN
  IF kill_type is HIT
    IF special is HEADSHOT: add_experience(killer, cfg exp "headshot")
                            add_bonus_money(killer, cfg money "headshot", HEADSHOT)
    IF special is EYESHOT:  same, with "eyeshot"
    IF special is BACKSTAB: add_bonus_money(killer, cfg money "backstab", BACKSTAB)
    IF special is none AND the weapon is a knife
                            add_bonus_money(killer, cfg money "knife_kill", KNIFEKILL)
  IF killer.streak > 0
    add_bonus_money(killer, cfg money "<streak>_kill_in_row", KIR, streak)
```

**Notes** — the streak bonus looks up a configuration key **built from the streak number**,
so the shipped data decides how long a streak has to be before it pays anything: a key that
does not exist yields zero. That is an elegant arrangement — the reward curve is entirely in
data, with no length limit compiled in — and it is the reason the lookup happens on every
kill rather than at thresholds.

Bonuses carry their reason as an enumerated kind, which reaches the client so it can name the
award on screen. Money and experience are separate currencies: a headshot pays both, a
backstab pays only money.

Only hit kills earn special bonuses. Bleeding and radiation deaths pay the base rival amount
and nothing else.

### `GetWinningPlayer`

**Contract** — the client with the highest frag count, or nothing on an empty server. Scans
every client including spectators.

**Notes** — the seed is -10000, which is a different arbitrary floor from the champion scan's
-100. Neither is derived; both are "low enough".

## Damage and respawn protection

### `OnPlayerHitPlayer` / `OnPlayerHitPlayer_Case`

**Contract** — intercepts a hit between two players before it is applied. Reads the hit
description out of the message, stamps the attacker's identity into it, lets the mode adjust
it, records who hit whom and with what if any damage survives, and writes the adjusted
description back into the message for the rest of the chain.

```text
FUNCTION on_player_hit_player(hitter_id, hitted_id, message)
  resolve both server objects and both player records ; RETURN if any is missing
  IF the target is not an actor: RETURN
  hit = read hit description from message
  hit.who = hitter.game_id
  adjust_hit(hitter, hitted, hit)              # overridable
  IF hit.power > 0
    hitted.last_hitter = hitter.game_id
    hitted.last_hit_weapon = hit.weapon
  write hit description back into message

FUNCTION adjust_hit(hitter, hitted, hit)        # the deathmatch rule
  IF hit.type is not a physical strike AND the target is invulnerable
    hit.power = 0 ; hit.impulse = 0
```

**Invariants** — the hit is **rewritten in place in the message**, so every later reader sees
the adjusted values. That is what makes the protection authoritative rather than advisory.

**Notes** — respawn protection blocks damage but **not physical strikes**. A protected player
can still be shoved by an explosion or a vehicle, which keeps them from being an immovable
obstacle in a doorway. The zeroed impulse in every other case is the second half of the same
thought: a protected player is not knocked about by gunfire either.

Recording the last hitter only when damage survived means a shot absorbed by protection does
not make the shooter the victim's killer if they later bleed out.

### `check_InvinciblePlayers` / `check_Player_for_Invincibility` / `OnPlayerFire`

**Contract** — respawn protection expires on a timer, and *also* the moment its holder
fires a weapon. The sweep clears the flag on any living player whose respawn is older than
the configured block time, and forces a synchronisation if any flag changed.

```text
FUNCTION check_player_for_invincibility(player)
  IF player.respawn_time + damage_block_time has passed
     AND player is invulnerable
    clear the invulnerable flag

FUNCTION on_player_fire(message)
  player = the firing player ; RETURN if absent or skipped
  IF player is invulnerable
    clear the flag ; signal_synchronize()
```

**Invariants** — **firing forfeits protection immediately.** Without it the protection would
be a free first shot, which is exactly the spawn-camping advantage it exists to prevent. A
rebuild that keeps the timer and drops this rule inverts the feature's purpose.

The sweep skips permanently dead players, whose flags are meaningless until they respawn.

## Respawn and spawn placement

### `OnPlayerReady`

**Contract** — the meaning of the ready signal depends on the phase. In the pending phase it
**toggles** the player's ready flag. In progress it is a request to re-enter play, honoured
only for a permanently dead, non-spectating, non-skipped player: the player is respawned, is
given their purchased weapons, and is tested for the clear-run bonus.

```text
FUNCTION on_player_ready(client)
  IF phase is PENDING
    toggle the ready flag ; signal_synchronize()
    RETURN
  IF phase is IN_PROGRESS
    IF the player is skipped, alive, or a spectator: RETURN
    IF this is the server's own client and spectator mode is on
      follow the next active player instead ; RETURN
    respawn(client, allow spectator body)
    IF the new body is an actor
      spawn the purchased weapons on it
      check_for_clear_run(player)
```

**Notes** — one key doing two unrelated things — "I am ready" before the round and "put me
back in" during it — is how the client gets away with a single spawn key. A rebuild may split
them; the phase test is the whole of the distinction.

### `RespawnPlayer`

**Contract** — runs the base respawn, pays the per-respawn money amount, arms respawn
protection if a block time is configured, and gives the new body the player's backpack.

**Invariants** — the backpack is spawned unconditionally on every respawn. It is what a
corpse's items are transferred into on death, so a body without one cannot drop loot.

### `RespawnPlayerAsSpectator`

**Contract** — resets a client to the pre-round state: clears their player record and item
list, back-dates their death time, refills their default items, resets their money to the
team's starting amount, and spawns them a spectator body.

**Notes** — the death time is set about a second in the past rather than to now. Several rules
compare against "time since death", and a freshly reset player must read as having been dead
long enough to act immediately rather than being held by a delay they never earned.

### `check_ForceRespawn`

**Contract** — when a force-respawn interval is configured, revives any permanently dead,
non-spectating player whose death is older than that interval: refills their default items,
respawns them without a spectator body, spawns their weapons, and tests the clear-run bonus.

**Notes** — this is the answer to a player who dies and stops pressing anything. The
"no spectator body" flag matters: a forced respawn puts the player straight back into the
world rather than into the spectator camera they would otherwise have to leave manually.

### `assign_RP` / `RP_2_Use`

**Contract** — chooses a respawn point for a spawning entity. Spectators and non-actors fall
through to the base behaviour. For an actor: partition the living players into friends and
enemies, refill the free-point cycle if it is exhausted, score every free point by its
distance to the nearest living enemy, and draw randomly from the *farther half*.

```text
FUNCTION assign_rp(entity, player)
  team = rp_set_for(entity)                  # always 0 here; team modes override
  IF entity is a spectator or not an actor: RETURN base.assign_rp(...)

  partition living players into friends and enemies
      # a player counts as a friend only when the score list is non-empty,
      # so in plain deathmatch EVERYONE is an enemy
  IF free_rpoints[team] is empty
    refill it with every point index except last_rpoint[team]

  FOR EACH free point
    nearest = smallest squared distance to any living enemy's position
    candidates.append(point index, nearest)
  sort candidates ascending by nearest

  half = candidates.count / (enemies is empty ? 1 : 2)
  pick = half > 0 ? candidates.count - half + random(half) : 0
  last_rpoint[team] = free_rpoints[team][candidates[pick].index]
  remove that entry from free_rpoints[team]
  place the entity at that point's position and angle
```

**Invariants** — the previously used point is **excluded when the free list is refilled**, so
the same point is never used twice in a row while any other exists. That is the minimum
anti-spawn-camping guarantee, and it survives even when only two points exist.

**Notes** — the selection is "random within the farthest half", not "the farthest point". Two
different pressures: the half keeps you away from the fight, and the randomness inside it
keeps an attacker from predicting where you land. With no enemies alive the half becomes the
whole list and any point is equally likely.

Distances are compared squared, and the per-point starting distance of 10000 stands for "no
enemy anywhere". Both are cost decisions with no behavioural weight.

The scoring record carries a "frozen" flag that sorts frozen points last. **Nothing ever
constructs one with the flag set**, so that dimension of the ordering is dead. Some intended
notion of a temporarily unusable point was never wired up.

The team argument is always zero here; team modes override the selector to return a real
team's respawn set.

## Money

### `Money_SetStart`

**Contract** — sets a player's round money to their team's configured starting amount. Does
nothing if the team is not described in configuration.

### `GetMoneyAmount`

**Contract** — reads one named money amount from a team's configuration section, defaulting to
zero for an absent key. Every amount in the team record goes through it, so a partially
configured team is well defined rather than undefined.

### `OnTeamScore`

**Contract** — pays every player a round-result amount: the winning-team figure to members of
the scoring team and the losing-team figure to everyone else, each in a full or a "minor"
variant chosen by the caller. Skipped and not-yet-ready clients are excluded.

**Notes** — the minor variants exist so that a round won on a technicality (a time limit, a
forfeit) can pay less than one won outright. Plain deathmatch never calls this; the team
modes do.

### `Check_ForClearRun`

**Contract** — pays a bonus to a player who enters a life having bought nothing. Suppressed
during warm-up.

**Notes** — the clear-run bonus is a deliberate counterweight to the buy economy: taking the
free default loadout is rewarded with money, so a player who dies broke is not locked out of
the next round. The test is on the *last purchase amount* being exactly zero, so selling and
re-buying to the same total does not qualify.

### `CanChargeFreeAmmo`

**Contract** — true unless the ammunition section appears in the mode's not-free list, read
from configuration at creation.

**Notes** — the membership test is a **substring search over the whole list string**, not a
tokenised comparison. A section whose name is a prefix of another's is therefore wrongly
treated as not free. A rebuild should split the list on its separators and compare whole
names; the intent is plainly a set.

## Buying and loadout

### `OnPlayerBuyFinished`

**Contract** — applies a client's completed purchase. Destroys the player's current items,
clears their item list, reads the money delta and the item list out of the message, and
records both. A living player's weapons are spawned immediately; a dead player's are not,
because they will be spawned when they respawn. Ends by permitting the buy menu to be
reopened.

```text
FUNCTION on_player_buy_finished(client, message)
  player = player record for client
  actor  = the player's server object          # may be absent if permanently dead
  destroy all of the player's items
  clear the player's item list

  player.last_buy_amount = message.read_int32()     # the money delta, signed
  count = message.read_int16()
  FOR EACH of count items
    group = message.read_byte()
    index = message.read_byte()
    player.item_list.append((group << 8) OR index)

  IF the player is not permanently dead
    spawn_weapons_for_actor(actor, player)
  allow the buy menu to be reopened
```

**Invariants** — the item encoding is **a packed pair of bytes in one sixteen-bit value**:
the low byte is the item's index in the mode's price table, the high byte is the weapon's
addon flags (scope, launcher, silencer). That packing is the loadout representation
everywhere in the session — in the player record, in the default-items list, in the spawn
path — so a rebuild must either keep it or change all of them together.

The money delta is signed and is applied *later*, when the weapons are spawned, not here.
That is what lets a dead player revise their purchase repeatedly without being charged each
time.

**Notes** — the message is trusted. The server destroys what the player had and rebuilds from
what the client sent, including the price the client claims to have paid. Everything
preventing a fabricated loadout lives on the client. A rebuild should price the list server
side from the same table the menu used.

Below the live implementation sits a much longer disabled version that reconciled the
player's *existing* inventory against the desired list — keeping matching items, adjusting
weapon addons in place, and destroying the rest. It was replaced by "destroy everything and
respawn it", which is simpler and costs a burst of entity creation on every purchase. The
helper it used (`CheckItem`) survives and is now unreachable.

### `SpawnWeaponsForActor`

**Contract** — spawns every item in the player's list onto their body, draining the list as it
goes, then charges the recorded purchase amount.

```text
FUNCTION spawn_weapons_for_actor(actor, player)
  IF the player's team is outside the loaded team list: RETURN
  WHILE the player's item list is not empty
    item = first entry
    spawn_weapon(actor, price_table.name_for(item low byte), item high byte, item list)
    remove the first entry
  IF NOT ignore_money_on_buy
    add_money(player, player.last_buy_amount)
```

**Invariants** — the list is drained, so the same purchase cannot be spawned twice. The spawn
call is handed the list as well, because an item that implies others — a weapon that brings
its ammunition — appends them.

**Notes** — charging *after* spawning, and only once, is the entire reason a dead player's
purchase is deferred. It also means a purchase that fails to spawn is still charged.

### `IsBuyableItem`

**Contract** — true when the named section appears in the mode's price table. This is the
gate on picking things up: an item the mode does not sell cannot be taken from the ground.

### `CheckItem`

**Contract** — reconciles one existing inventory item against a desired-items list: matches by
price-table index, tolerates or rejects an addon mismatch depending on the caller's exactness
flag, reconfigures the addons in place when tolerated, and otherwise marks the item for
destruction.

**Notes** — reachable only from the disabled purchase path described above. It is recorded
because the **addon-change event** it emits is the live mechanism elsewhere: changing a
weapon's attachments is a server-object flag update plus one broadcast event, not a respawn.

### `OnPlayer_Sell_Item`

**Contract** — empty. Deathmatch has no sell path; artefact hunt supplies one.

## Teams, skins and default items

### `LoadTeams` / `LoadTeamData`

**Contract** — points the price table at this mode's cost section, which must exist, and loads
a single team configuration section. Loading a team reads its skin list, its default item
list, and fifteen named money amounts plus the invulnerable-kill modifier.

**Notes** — the money amounts cover situations plain deathmatch never reaches — target
bonuses, round draws, rivals wiped out. They are read here because the team record is shared
with the modes that do use them, and reading them unconditionally means a team section can be
written once and used by any mode.

The invulnerable-kill modifier defaults to **0.5** when the key is absent: a kill made under
respawn protection pays half. It is the only money value with a non-zero default, which marks
it as a rule rather than a tuning knob.

### `LoadSkinsForTeam` / `LoadDefItemsForTeam`

**Contract** — read a team's comma-separated skin names and default item names. Default items
are converted to price-table indices at load, so the runtime never looks an item up by name.

**Notes** — an item name absent from the price table converts to the table's "not found"
marker truncated into the index field, which silently becomes a valid-looking index. A
rebuild should reject unknown names at load.

### `SetSkin`

**Contract** — assigns a visual model to a spawning entity: the team's configured skin at the
requested index, that team's first skin when the index is out of range, and one of three
hardcoded fallback models when the team has no skins configured at all. The assembled path
must fit in 64 characters.

**Notes** — the three fallback model names are the shipped defaults for the neutral, first and
second teams. They exist so a level with no skin configuration still runs, and a rebuild can
drop them if it makes the configuration mandatory.

The 64-character limit is a real constraint: the visual name is a fixed-width field in the
spawn record.

### `OnPlayerSelectSkin` / `OnPlayerChangeSkin`

**Contract** — a skin request is applied, acknowledged to the requesting client with a menu
response message, and broadcast through a state synchronisation. Applying it clears the
player's spectator flag and **kills them**, so the new model takes effect immediately.

```text
FUNCTION on_player_change_skin(client, skin)
  player.skin = skin
  clear the player's spectator flag
  IF skin is "any": player.skin = random index into the team's skin list
  kill the player                       # forces a respawn with the new visual
```

**Invariants** — changing skin costs a life. The visual is baked into the spawn record, so
there is no way to change it on a live body.

**Notes** — the code contains a guard that compares the requested skin against the field it
has just written to, so it always matches and always returns early. The effect is that the
random-skin path is **unreachable** and the kill never happens on this path — the skin is
recorded and takes effect at the player's next natural death. Whether the early return was
meant to sit before the assignment is not recoverable from the source.

## Item flow

### `OnTouch`

**Contract** — decides whether a player may take a world item. Weapons are allowed unless the
player already holds one in the same slot; ammunition, grenades and outfits are always
allowed; anything the mode sells is allowed; everything else is refused. A dropped player
bag is a special case: it is not picked up but *emptied* into the toucher and destroyed.

```text
FUNCTION on_touch(actor_id, item_id, forced) -> allowed
  IF the toucher is not an actor: RETURN false
  IF the item is a weapon
    IF the actor already holds a weapon in the same slot
      IF forced: reject that weapon and re-issue the take, so the new one replaces it
      RETURN false
    RETURN true
  IF the item is ammunition, a grenade or an outfit: RETURN true
  IF the item is a player bag lying on the ground
    FOR EACH child of the bag
      IF on_touch(actor, child, not forced) is false: reject it out of the bag
      ELSE transfer it from the bag to the actor
    send the transfers as one packed event
    destroy the bag
    IF pda_hunt: pay the looter the configured bonus
    RETURN false
  IF the item is sold by this mode: RETURN true
  RETURN false                              # unknown: refuse, for safety
```

**Invariants** — one occupied slot, one weapon. The forced variant is how a deliberate swap
works: reject the held weapon and take the new one, as two events in order.

The transfers out of a bag are batched into a single packed event rather than sent
individually. With a full loadout that is a dozen ownership changes, and sending them
separately would let a client render the actor holding a partial inventory for a frame.

**Notes** — refusing anything unrecognised is the right default here: the alternative is a
player picking up a level prop and carrying it into the round.

The bag bonus is the "PDA hunt" rule — looting a dead opponent pays, which is what makes
walking over to a body worthwhile.

### `OnDetach` / `FillDeathActorRejectItems`

**Contract** — when a player's bag is detached, it is filled from the player's inventory:
every item the mode sells is transferred into it, except the outfit, except anything on the
reject list, while the knife and the torch are destroyed outright. The reject list holds
whatever the player was actively holding at the moment of death, excluding the knife.

**Invariants** — **the weapon in the dying player's hands does not go into the bag.** It is
rejected instead, which drops it where they fell. That separation is the visible rule: the
weapon you were using lands next to your body, the rest of your kit stays in the bag.

**Notes** — the knife and the torch are destroyed rather than dropped because every player
gets them free, so dropping them would litter the level with items nobody needs. The outfit
is excluded for the same reason from the other direction — it is worn, not carried.

### `OnDestroyObject`

**Contract** — when an object disappears: if it was the server's spectator target, follow
someone else; remove it from the corpse list; if it was a live player's actor body during a
round, put that player into a spectator body; and tell the item respawner.

**Notes** — the spectator body is spawned here rather than at death, so a player whose corpse
is removed for any reason — a round end, a timeout, a cleanup — always ends up somewhere they
can see from.

### `RemoveItemFromActor` / `on_death`

**Contract** — destroy one item by broadcast event; and forward a death to the victim's own
server object so it can run its own death handling.

## Connection

### `OnPlayerConnect`

**Contract** — prepares a connecting client. A genuinely new connection has its player record
cleared, its team set to zero and its skin marked unselected; a reconnection keeps its
record. Either way the player starts flagged as a spectator with the skip flag cleared. The
server's own client on a dedicated server, or in spectator mode, is marked skip and takes no
further part. New connections get their starting money and their default items.

**Invariants** — a **reconnecting** player keeps their money, frags and rank. That is the
rule that makes a dropped connection survivable, and it is the only thing the reconnect flag
does here.

### `OnPlayerConnectFinished`

**Contract** — spawns the client a spectator body, forces their team to zero and their
spectator and ready flags on, broadcasts a player-connected message carrying their exported
state, marks them network-ready, and sends them the current anomaly states.

**Invariants** — the new player is marked **ready** on arrival. Without it a lobby would stall
on a player who has connected but not yet chosen anything; with it, they are counted as ready
until they act.

**Notes** — the anomaly states are pushed at connection rather than requested, because the
zones spawned with the level and their live/disabled state is session state, not level data.
A client that missed it would see every anomaly active.

## Anomaly rotation

### `LoadAnomalySets`

**Contract** — reads the named anomaly sets out of the *level's* configuration, under a
mode-specific base section: up to twenty sets named in sequence, each a comma-separated list
of zone names, plus a list of permanently active zones. Missing sets are skipped, not fatal.

**Notes** — the sets live in the level's own configuration rather than in the game
configuration, because which zones exist is a property of the map. The twenty-set cap is a
compiled-in scan bound with no reason in the source; a rebuild should scan until a gap.

The base section name is virtual, so each mode reads its own sets from the same level.

### `OnPreCreate` / `OnCreate` / `OnPostCreate`

**Contract** — three hooks around zone creation. Pre-create decides whether a zone may spawn
at all. Create applies the zone's configured maximum starting power if one is authored.
Post-create resolves a spawned zone's name against the loaded sets, records its identifier in
the matching set, and switches it off.

```text
FUNCTION on_post_create(entity_id)
  zone = the entity, if it is a custom zone with no owner ; RETURN otherwise
  FOR EACH set index
    IF the zone's name appears in that set
      anomaly_ids[set].append(entity_id)
      broadcast zone state change(entity_id, DISABLED)
      RETURN
```

**Invariants** — every zone that belongs to a set spawns **disabled**. The live set is then
switched on explicitly, so the sequence is "spawn everything off, turn one group on" rather
than "spawn on, turn the rest off". A rebuild reversing that will flash every anomaly in the
level for the first frame of every round.

A zone with an owner is skipped: it belongs to an artefact activation or a script, not to the
rotation.

**Notes** — the pre-create filter is now a **constant yes**. Its disabled body would have
refused to spawn any zone not named in some set, so that a level's unlisted anomalies simply
did not exist. As it stands they spawn and stay enabled forever, because nothing ever names
them. Whether that is the intended behaviour or a regression is not recoverable.

### `StartAnomalies` / `check_for_Anomalies`

**Contract** — rotation. The live set is switched off, a new one is chosen — from an explicit
argument, or drawn from a shuffle bag that refills excluding the current set — and switched
on. The periodic check rotates when the configured set length has elapsed.

```text
FUNCTION start_anomalies(requested)
  IF there are no sets: RETURN
  IF unused_set_ids is empty
    refill it with every set index except live_set_id
  draw a random entry from unused_set_ids and remove it

  IF live_set_id names a real set
    send zone state DISABLED for every zone in it
  IF requested is given
    live_set_id = requested ; unused_set_ids.clear()
  ELSE
    live_set_id = the drawn entry
  IF anomalies_enabled
    send zone state IDLE for every zone in the live set
  set_started_at = server_time

FUNCTION check_for_anomalies() -> bool
  IF NOT anomalies_enabled: RETURN false
  IF a set is already live
    IF anomaly_set_length is 0: RETURN false          # 0 means "never rotate"
    IF set_started_at + anomaly_set_length has not passed: RETURN false
  start_anomalies() ; RETURN true
```

**Invariants** — the shuffle bag guarantees every set is used once before any repeats, and
excludes the current set when refilling so the same set is never chosen twice running. Same
shape as the respawn-point cycle, and for the same reason.

**Notes** — "no set is live" is encoded as the identifier **1001**, tested against a threshold
of **1000**. The real identifiers are small set indices, so any value above the threshold
means none. Two magic numbers standing in for an optional; a rebuild should use one.

An explicit request clears the shuffle bag, so a scripted set change restarts the rotation
rather than continuing the cycle.

### `Send_EventPack_for_AnomalySet` / `Send_Anomaly_States`

**Contract** — both batch a run of per-zone state-change events into a single packed message,
each inner event preceded by its length as one byte. The first broadcasts one set's zones to
one state; the second sends *every* set's current state to one client, live set idle and the
rest disabled.

**Invariants** — the packed form is a frozen wire layout: a message header followed by
repeated (one-byte length, event bytes) pairs. The one-byte length caps an inner event at 255
bytes, which a zone state change is comfortably under.

**Notes** — the state sweep abandons the whole message on the first empty set rather than
skipping it, so a level with a gap in its set numbering sends an incomplete picture. With
sets loaded contiguously this cannot happen, which is why it has never mattered.

### `Is_Anomaly_InLists`

**Contract** — constant yes.

**Notes** — its disabled body is the membership test the pre-create filter was written for:
a zone belongs if it has an owner, is named in the permanent list, or is named in any set.
The permanent list is loaded and, with this test disabled, is never read.

## The server's own spectator mode

### `SM_SwitchOnNextActivePlayer` / `SM_SwitchOnPlayer` / `net_Relcase`

**Contract** — a non-dedicated server can run without playing, its window following live
players and switching between them on a timer. The switch picks a random living, non-skipped
player; with none available it falls back to the server's own body. Switching moves the
view, turns the previous target's first-person item display off and the new target's on, and
arms the next switch. The relocation hook clears the target when it is destroyed.

**Invariants** — the candidate must be alive **and holding something**; a player with no
active item is skipped, because the point of the mode is watching them fight.

**Notes** — turning the first-person item display on for the watched player is what makes the
view look like that player's own screen rather than a camera at their eyes. It must be turned
off on the way out or two players' weapons render at once.

The view target is a raw reference, which is why it needs a destruction hook at all.

## Session state export

### `net_Export_State`

**Contract** — appends the mode's join-time state after the base session's.

```text
frag_limit            : int (32-bit, signed)
time_limit            : int (32-bit, signed)     # minutes
force_respawn         : int (32-bit)             # seconds
warmup_deadline       : int (32-bit)             # server clock, 0 when none
damage_block_indicators : byte
team_count            : int (16-bit)
  team_count times: a raw TeamScore record       # see Invariants
# only in the PLAYER_SCORES phase:
winning_player_name   : zero-terminated text
```

**Invariants** — the team scores are written as a **raw memory image of the score record**,
not field by field. That freezes the record's layout — a signed score and a sixteen-bit
target count, with whatever padding the compiler inserts — into the wire protocol. It is the
one place in this file where a rebuild cannot simply choose its own representation: it must
reproduce the exact bytes, or change both ends together.

The winner's name is present **only in the scores phase**, so the message is not
self-describing; the reader must know the phase, which it does because the base session's
state carries it.

**Notes** — the damage-block *time* is not sent, only the indicator flag. The client is told
whether to draw protection markers but not how long protection lasts, which it does not need
because the flag on each player record carries the fact.

### `GetTeamScore` / `SetTeamScore` / `GetNumTeams`

**Contract** — indexed access to the team scores, with the setter refusing outside the
in-progress phase so that a scoring event arriving during the end-of-round dwell cannot
change a decided result.

### `WriteGameState`

**Contract** — appends this mode's fields to the round statistics record: whether warm-up is
running, whether anomalies are enabled, the leading player's name, the two limits, and the
round's elapsed seconds. The warm-up flag, the best killer and the elapsed time are omitted
from the per-round-result variant, which describes a round that has already ended.

## Script surface

### `type_name`

**Contract** — the mode's name, `deathmatch`. It is what the session advertises and what
scripts compare against.

## Dead and unreachable

**Contract** — several declared units do nothing.

- `LoadItemRespawns` — empty and never called. Item respawning is the base session's.
- `ConsoleCommands_Create` / `ConsoleCommands_Clear` — empty. Deathmatch registers no
  commands of its own; its settings reach the console through the base.
- `OnTeamsInDraw` — empty and never called from anywhere.
- `OnPlayerDisconnect` — pure delegation.
- `RP_2_Use` — always zero; exists to be overridden.
- `OnRender` — debug only, entirely commented out. It drew pathfinder routes between every
  pair of respawn points, which is how the respawn placement was originally validated.
- An unused slot constant is defined at the top of the file and referenced nowhere.
