# src/xrGame/game_cl_artefacthunt.cpp

> Artefact hunt on the client: one artefact on the map, who is carrying it, where it shows up, and the paid-respawn reinforcement window between waves.

**Needs** — [`game_cl_artefacthunt.h`](game_cl_artefacthunt.h.md) · [`game_cl_teamdeathmatch.h`](game_cl_teamdeathmatch.h.md) · [`UIGameAHunt.h`](UIGameAHunt.h.md) · [`game_cl_artefacthunt_snd_msg.h`](game_cl_artefacthunt_snd_msg.h.md) · [`Artefact.h`](Artefact.h.md) · [`Actor.h`](Actor.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`Inventory.h`](Inventory.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`Level.h`](Level.h.md) · [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [`ui/TeamInfo.h`](ui/TeamInfo.h.md) · [`ui/UIMessageBoxEx.h`](ui/UIMessageBoxEx.h.md) · [`ui/UISkinSelector.h`](ui/UISkinSelector.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md) · [`ui/UIMainIngameWnd.h`](ui/UIMainIngameWnd.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrServerEntities/clsid_game.h`](../xrServerEntities/clsid_game.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)

**Used by** — reached through its declarations in [`game_cl_artefacthunt.h`](game_cl_artefacthunt.h.md); callers name that, not this file.
**Tier floor** — T2: snapshot decoding, map bookkeeping and timed screen state

## Purpose

Artefact hunt is team deathmatch with an objective: a single artefact spawns somewhere in the
level, and a team scores by carrying it to their base. Everything about teams, skins, buying
and readiness is inherited; this file adds four things.

1. **Tracking one object across the whole session.** The artefact's identity, who holds it
   and which team that is, arrive in every snapshot and drive the map marker, the minimap
   and the sprint restriction.
2. **A narrated objective.** Taken, dropped, delivered, spawned, destroyed — each with a
   feed message and a spoken line chosen from *three* variants depending on whether it was
   you, your team, or the enemy.
3. **Reinforcement waves.** Respawning is not continuous: the server publishes a wave
   interval and the time to the next wave, and a dead player waits for it.
4. **Paying to skip the wave.** A player with enough money can spawn immediately, at a
   configured cost, through a confirmation box.

## State

```text
RECORD ArtefactHuntClient EXTENDS TeamDeathmatchClient
  artefacts_num          : int (8-bit)    # how many the leading team still needs; the "frag limit"
  artefact_id            : int (16-bit)   # the live artefact entity, 0 for none
  artefact_bearer_id     : int (16-bit)   # who carries it, 0 for nobody
  team_in_possession     : int (8-bit)
  old_artefact_id        : int (16-bit)   # previous-snapshot copies, used only to
  old_artefact_bearer_id : int (16-bit)   #   clean up a map marker for a vanished artefact
  old_team_in_possession : int (8-bit)
  reinforcement_interval : int (ms)       # 0 means continuous respawn
  next_reinforcement_at  : int (ms)       # absolute server time, or 0
  spawn_cost             : int (money)    # negative; default -10000
  artefact_spawn_effect  : text           # particle effect names, optional
  artefact_vanish_effect : text
  bearer_cannot_sprint   : bool           # server policy, per snapshot
```

Invariants:

- **`artefact_id` of zero means no artefact exists.** Zero is a sentinel, not an entity, and
  every reader tests it first.
- **`next_reinforcement_at` is absolute server time**, converted from the relative value on
  the wire at decode. That conversion has to happen at decode or the deadline drifts by the
  age of the snapshot.
- **A bearer implies an artefact**, and the map pass relies on it.

## `net_import_state`

**Contract** — decodes the mode's snapshot after the inherited team deathmatch state: the
target count, the artefact and its bearer, the possessing team, the sprint policy, and the
reinforcement pair.

```text
FUNCTION net_import_state(packet)
  base.net_import_state(packet)
  artefacts_num        = packet.read_byte()
  artefact_bearer_id   = packet.read_int16()
  team_in_possession   = packet.read_byte()
  artefact_id          = packet.read_int16()
  bearer_cannot_sprint = packet.read_byte() != 0

  reinforcement_interval = packet.read_int32()
  IF reinforcement_interval > 0
    next_reinforcement_at = packet.read_int32() + server_time     # relative -> absolute
  ELSE
    next_reinforcement_at = 0
```

**Invariants** — the second reinforcement field is present on the wire **only** when the
interval is positive. This is the one conditional field in the snapshot and both ends must
agree on the rule, or every field after it is misread.

**Notes** — the deadline arrives relative and is stored absolute against the server clock,
not the local one. That is what makes the countdown correct despite latency: the client
never has to know how old the snapshot is.

## `TranslateGameMessage`

**Contract** — renders the five objective events into the feed and plays the matching
announcement. Anything else falls through.

```text
FUNCTION translate_game_message(event, packet)
  SWITCH event
    ARTEFACT_TAKEN:
      read player, team ; feed "<player> has taken the artefact"
      announce(taken_by_me | taken_by_my_team | taken_by_enemy, indexed by MY team)
    ARTEFACT_DROPPED:
      read player, team ; feed "<player> has dropped the artefact"
      announce(artefact lost)
    ARTEFACT_ONBASE:
      read player, team ; feed "<team> scores"
      announce(scored_by_me | scored_by_my_team | scored_by_enemy, indexed by MY team)
    ARTEFACT_SPAWNED:
      feed "a new artefact has appeared" ; announce(new artefact)
    ARTEFACT_DESTROYED:
      read artefact id ; play the vanish effect at its position if it still exists
      feed "the artefact was destroyed"
```

**Notes** — the three-variant announcements are the design point. Which line plays depends
on the listener's relationship to the event (I did it, my team did it, the enemy did it) and
on *which team the listener is on*, because each team's lines are spoken in its own voice.
That is six recordings per event, selected by adding the listener's zero-based team to a
base identifier. A rebuild should keep the two-dimensional selection and not flatten it.

The drop and destroy events have no per-team variants — losing the artefact sounds the same
to everyone.

## `shedule_Update`

**Contract** — the per-update screen pass: the buy prompt, the context-dependent
press-a-key prompt, the reinforcement countdown, the score, and closing the paid-spawn box
when it becomes irrelevant.

```text
FUNCTION scheduled_update(delta)
  base.scheduled_update(delta)
  IF dedicated or no game ui: RETURN
  clear both prompts

  IF phase is IN_PROGRESS
    buy_enabled = on base AND not permanently dead
    IF I am controlling an ACTOR                     # playing
      IF buy_enabled and no menu is up: prompt "press B to buy"
      IF permanently dead: prompt "press fire for spectator"
    ELSE                                              # spectating, waiting to spawn
      IF team and skin are chosen
        IF reinforcement waves are on
          IF the paid-spawn box is closed AND I can afford the spawn
            prompt "press jump to pay for a spawn"
        ELSE
          prompt "press jump to spawn"
      ELSE
        prompt "press jump to select a team" (or a skin)

    IF a reinforcement deadline exists AND there is a view entity AND warm-up is over
      seconds_left = ceiling((deadline - server_time) / 1000), floored at 0
      show (seconds_left, interval)
    ELSE
      show (0, 1)                                    # see Notes
    set score

  IF phase is TEAM1_ELIMINATED or TEAM2_ELIMINATED
    caption "<team> eliminated" ; set score

  IF the paid-spawn box is shown AND (the round is over OR I am not permanently dead)
    hide it
```

**Notes** — the "show (0, 1)" fallback feeds a zero remaining against an interval of one, so
the countdown display reads as complete rather than as an empty division. A rebuild with a
real "no countdown" state should use it instead.

The paid-spawn box is closed reactively, by testing every update whether it is still
relevant, rather than by whoever made it irrelevant. Same robust shape as the team-select
window in team deathmatch.

The team-and-skin prompt branch has a dangling conditional in the source — the skin prompt
is attached to the wrong test and is unreachable. The *intent* is clear and a rebuild should
implement it: prompt for whichever of team or skin is missing.

The buy prompt is written twice per update, cleared and then possibly set, which is how a
prompt with several sources is kept from sticking.

## `PlayerCanSprint`

**Contract** — the artefact bearer may not sprint, when the server says so. Everyone else
always may.

```text
FUNCTION player_can_sprint(actor) -> bool
  IF there is no bearer: RETURN true
  IF bearer_cannot_sprint AND actor is the bearer: RETURN false
  RETURN true
```

**Notes** — this is the objective's entire mechanical weight: carrying the artefact makes you
slow, so a delivery is a team effort rather than a sprint. The restriction is server policy
in the snapshot, so a match can turn it off.

The flag is stored in a *file-scope* variable rather than a member, which makes it global to
the process. With one mode instance at a time that is harmless; it is nevertheless the kind
of state a rebuild should put on the mode.

## `UpdateMapLocations` / `GetMapEntities`

**Contract** — the artefact gets exactly one map marker, of one of three kinds: neutral
(lying loose), friendly (carried by my team) or enemy (carried by theirs). The marker is
replaced, not accumulated, when the situation changes. The minimap pass adds one coloured
point: white for a loose artefact, yellow for a friendly bearer, red for an enemy bearer.

```text
FUNCTION update_map_locations()
  base.update_map_locations()
  IF no local player: RETURN
  IF no artefact
    IF there was one last snapshot: remove every marker for it
  ELSE IF no bearer
    IF no neutral marker: remove all markers for it ; add neutral ; enable its pointer
  ELSE IF the bearer's team is mine
    IF no friendly marker: remove all markers for it ; add friendly ; enable its pointer
  ELSE
    IF no enemy marker:    remove all markers for it ; add enemy   ; enable its pointer
  remember artefact, bearer and team for the next pass
```

**Invariants** — the "remove all markers for this object, then add one" shape is what
guarantees exactly one marker survives a transition. Testing for the *target* kind before
doing anything makes the pass idempotent, so it can run every update.

**Notes** — the previous-snapshot copies exist solely for the first branch: when the artefact
is destroyed, its identifier becomes zero and there is nothing left to remove the marker
*by*, so the old identifier is kept for exactly that purpose. That is the only use of the
three "old" fields, and a rebuild that removes markers on the destroy event needs none of
them.

The enemy branch in the source wraps a disabled block that would have *hidden* the artefact
from the enemy team's map while its bearer wore a particular outfit and stood still —
effectively a stealth suit. It is commented out, so the artefact is always visible to both
teams. The intent is recorded here because it is a real design that shipped disabled; the
live rule is simpler, and simpler is what a rebuild should implement.

The minimap pass returns a marker for the bearer's position rather than the artefact's, and
falls through to nothing when the artefact is held by an entity that is not a known player —
for instance while it is in flight between hands.

## `NeedToSendReady_Spectator`

**Contract** — decides what the spawn key does while spectating. In the pending phase, fire
readies. In progress, jump readies — *unless* reinforcement waves are on and the player can
afford to pay, in which case the confirmation box is shown instead and nothing is sent.
During warm-up the wave rule is skipped entirely.

```text
FUNCTION need_to_send_ready(key, player) -> bool
  ready = (phase is PENDING and key is fire)
       OR (phase is IN_PROGRESS and key is jump and can_be_ready())

  IF phase is IN_PROGRESS and key is jump and warm-up is running: RETURN ready

  IF phase is IN_PROGRESS and key is jump
     AND reinforcement waves are on
     AND the paid-spawn box is not already shown
     AND my money plus the (negative) spawn cost is still non-negative
    compose the confirmation text with my money and the cost
    IF team and skin are chosen: show the box
    RETURN false                             # do not send; the box will
  RETURN ready
```

**Notes** — the affordability test adds a *negative* cost and checks the result is
non-negative, which is the same as "money is at least the cost". Written that way because
the configured value is stored negative, as a delta to be applied. The confirmation text is
built from the absolute values of both, so the player reads two positive numbers.

The warm-up escape means waves do not apply before the round starts: everybody spawns freely
while warming up.

Note that the confirmation box is shown even when team or skin are not yet chosen — only the
*showing* is gated, while the early return is not. A player in that state gets no box and no
spawn, which is the dead end the prompt in the update pass is meant to steer him out of.

## `OnBuySpawnMenu_Ok`

**Contract** — the player confirmed paying for a spawn. Sends one event from the player's
current entity; the server charges and spawns.

**Notes** — no money is deducted locally and no state is set. The client asks and waits for
the next snapshot, which is right for anything the server must arbitrate.

## `CanCallBuyMenu` / `CanBeReady`

**Contract** — the buy menu additionally requires the round in progress, the player's data
ready, no other window shown, and — the artefact-hunt-specific part — a **living actor**.
`CanBeReady` builds the skin and buy menus, then refuses and opens the relevant window if
team or skin is missing.

**Notes** — `CanBeReady` deliberately does **not** delegate to team deathmatch's version (the
call is present and commented out). The reason is the buy step: once team and skin are
settled it clears an unshown buy menu and reports ready, where team deathmatch would add
further conditions. A rebuild should notice that artefact hunt's readiness is its own rule,
not an extension.

## `OnSellItemsFromRuck`

**Contract** — sells everything in the player's backpack in one event. Refused unless the
player is alive and standing on his base.

```text
FUNCTION on_sell_items_from_ruck()
  IF no local player OR permanently dead OR not on base: RETURN
  actor = my entity ; IF none: RETURN
  event = new event(PLAYER_ITEM_SELL, destination = actor)
  event.write_int16(count of backpack items)
  FOR EACH item IN backpack: event.write_entity_id(item.id)
  send(event)
```

**Notes** — one event carrying every identifier, rather than one event per item. That is the
right shape for an operation the server must apply atomically — a partial sale is not a
state the game has a name for.

## `SetScore`

**Contract** — the team scores from team deathmatch, plus the watched player's rank and his
frag count against the artefact target.

**Notes** — the "frag limit" display is repurposed here to show the artefact target, so the
same widget reads as "frags / artefacts needed". That is why the target count is in the
snapshot at all.

## `Init` / `createGameUI` / `SetGameUI` / `OnConnected` / `getTeamSection` / `GetBaseCostSect`

**Contract** — load both teams' data, read the two optional particle effect names and the
spawn cost from the mode's configuration section, instantiate the mode's screen by class
identifier, and re-resolve the cached screen handle wherever it might have been replaced.

**Notes** — the file also carries a large disabled block that would have read the level's
authored spawn-point data and started a permanent particle effect at each team's base,
configured per team. It is commented out, so team bases have no visual effect of their own in
this mode. Recorded because the mechanism — level-authored points carrying a type and a
per-mode filter, driving presentation — is real elsewhere in the engine.

## `OnSpawn` / `OnDestroy` / `SendPickUpEvent`

**Contract** — a spawning artefact plays the configured appearance effect at its position.
Destruction adds nothing. The pick-up event deliberately bypasses both intermediate modes
and goes straight to the base game state's implementation.

**Notes** — that bypass is load-bearing and easy to lose: team deathmatch or deathmatch may
add pick-up restrictions, and artefact hunt must not apply them, because taking the artefact
is the objective. Skipping two levels of the hierarchy is how the original says so.

## `LoadSndMessages`

**Contract** — loads fourteen announcements: a new artefact, the artefact lost, and then two
teams times three perspectives (me, my team, the enemy) for each of "taken" and "delivered".
