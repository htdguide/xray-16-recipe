# src/xrGame/game_cl_deathmatch.cpp

> Deathmatch on the client: the warm-up countdown, the frag and time limits, the sequence a player walks through before he can spawn, and the vote display.

**Needs** — [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`UIGameDM.h`](UIGameDM.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Spectator.h`](Spectator.h.md) · [`Weapon.h`](Weapon.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`map_manager.h`](map_manager.h.md) · [`ui/UISkinSelector.h`](ui/UISkinSelector.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md) · [`ui/UIVote.h`](ui/UIVote.h.md) · [`game_cl_deathmatch_snd_messages.h`](game_cl_deathmatch_snd_messages.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`clsid_game.h`](../xrServerEntities/clsid_game.h.md)
**Used by** — [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md)
**Tier floor** — T2: decodes an extended snapshot and drives timed screen state

## Purpose

The rules of deathmatch as the client sees them, and the template for every other mode. Three
things in it are the real content: the **entry sequence** a player must complete before he can
spawn, the **warm-up** that separates practice from scoring, and the **per-tick screen
composition** that turns match state into captions.

## State

Declared in [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md). What this file establishes:

```text
  frag_limit     : int          # 0 means none
  time_limit     : int          # authored in minutes, stored in milliseconds
  force_respawn  : int          # authored in seconds, stored in milliseconds
  warmup_until   : int          # server time; 0 means the warm-up is over
  damage_block_indicators : bool  # draw a marker over protected players
  teams          : list<TeamState>
  winner_name    : text
  skin_selected, spectator_selected, menu_called_from_ready, first_run : bool
```

**Invariants** — the time limit and respawn delay are **converted from the authored unit on
receipt**, so everything downstream works in milliseconds. The warm-up is an absolute server
time, not a duration.

A warm-up in progress makes two things true at once: the buy screen ignores money and rank,
and rewards are suppressed. Practice is free and worthless.

## `net_import_state`

**Contract** — extends the base snapshot with the frag limit, the time limit, the force-respawn
delay, the warm-up deadline, the damage-indicator flag, and the team score list. In the
scoring phase it also reads the winner's name and, if that is the local player, plays the
victory announcement.

**Invariants** — the team score records are read as a **raw memory block**, not field by
field. That freezes the record's layout into the wire format and makes the snapshot
byte-order- and padding-dependent. A rebuild must serialize the two fields explicitly and will
then not match the original's bytes.

This is the pattern every derived mode repeats: the snapshot is a chain of decoders, each
reading the base's fields and then its own, with no length prefix between them. A mode that
gets its own field count wrong desynchronises every mode below it.

## the entry sequence — `CanBeReady`

**Contract** — deathmatch's answer to "the player pressed ready". It is a state machine over
the three things a player must do before spawning: choose a skin, choose a loadout he can
afford, and confirm.

```text
FUNCTION can_be_ready() -> bool
  RETURN false IF there is no local player
  mark that the menus were opened from a ready press
  make sure the skin screen and the buy screen exist
  IF the buy screen is not open THEN reset it to the player's default items

  IF no skin has been chosen THEN
    open the skin screen
    RETURN false                       # not ready yet; the skin screen will re-enter here

  IF a buy screen exists THEN
    affordable = the last-used preset is empty, OR its cost is within our money,
                 OR the warm-up is suppressing costs
    IF NOT affordable THEN
      open the buy screen
      RETURN false
    confirm the purchase and RETURN true

  RETURN true
```

**Invariants** — "not ready" is returned with a screen open, and the screen's own confirmation
re-enters the sequence: choosing a skin calls back into the ready path if the sequence was
started from a ready press, which is what the "called from ready" flag records. Without it,
opening the skin screen from its own key would spawn the player.

The **empty preset is affordable**: a player who wants to spawn with nothing may. Only a
non-empty preset he cannot pay for blocks him.

## the three gates

**Contract** — each answers whether a screen may be opened now.

- **Buy** — only while the match is in progress, the server has confirmed the last buy screen
  closed, the player is currently a *spectator* (that is, waiting to spawn), a skin has been
  chosen, he has not chosen to spectate permanently, and neither the skin screen nor the
  inventory is open.
- **Skin** — only while the match is in progress and neither the inventory nor the buy screen
  is open. Opening it also seeds it with the player's current skin.
- **Inventory** — only while the match is in progress, the controlled entity is a living
  actor, the skin screen is closed, and the player is not dead.

**Invariants** — the buy gate's requirement that the player be a spectator is the whole
economy rule: **you buy before you spawn, never after**. The server's buy-screen-closed
acknowledgement is what stops a second purchase before the first is applied.

Every tick closes any screen whose gate has since become false, which is how a screen left
open across a phase change is cleaned up.

## `shedule_Update` — the screen composition

**Contract** — per tick, clear every caption and then re-derive the ones the current phase
wants. Captions are never edited in place; the clear-then-set discipline is what keeps a stale
caption from surviving a state change.

While the match is in progress and the local player is valid:

- **Time remaining** — the time limit minus the elapsed phase time, clamped at zero, shown only
  when a limit is set and the warm-up is over.
- **Server information** — shown once on the first tick, unless this is a demo; the flag is
  cleared only when the screen actually appeared. The active vote is requested at the same
  moment.
- **Money** — taken from whoever the camera is on, not from the local player, so a spectator
  sees the money of the player he is watching.
- **Warm-up countdown** — above ten seconds, a formatted clock; between one and ten seconds, a
  bare count with an announcement **once per second boundary**; below one second, "go".
- **Prompts** — while the player is a spectator with no screen open, prompt to press jump
  (worded differently before and after a skin is chosen) and to press buy when the buy gate is
  open.
- **Spectator caption** — the spectator camera's own description of what it is looking at.
- **Vote progress** — time remaining and the fraction of players who agreed, recomputed by
  scanning the player table each tick.
- **Forced respawn** — a countdown for a dead player who is not a spectator.
- **Rank and frags** — taken from whoever the camera is on.

In the pending phase it shows the player list; in the scoring phase it also prints the winner.

**Invariants** — the per-second announcement uses a **function-local static** holding the last
whole second announced, so it is shared across every instance and survives a match. With one
match at a time it works; a rebuild keeps it as state.

**Notes** — the warm-up countdown announcement indexes the five countdown identifiers by
adding the seconds remaining minus one to the first, which is why that run must be contiguous
(see [`game_cl_deathmatch_snd_messages.h`](game_cl_deathmatch_snd_messages.h.md)).

The forced-respawn countdown subtracts the player record's death time from the delay. As noted
in [`game_base.cpp`](game_base.cpp.md), that field holds an *age* on the client, so the
arithmetic is "delay minus age" and happens to be right — the two defects cancel.

## `OnKeyboardPress` / `OnKeyboardRelease`

**Contract** — after the base, deathmatch handles: the scores key (hold to show the frag list,
release to hide), the inventory key (toggle, gated), the buy key (toggle, gated, resetting to
the default loadout when opening), and the skin key (toggle, gated). Demo playback admits only
the scores and crouch keys.

**Notes** — the map key's handler is commented out, so the map screen is unreachable in
multiplayer even though the mode maintains map markers for it.

## `OnVoteStart` — the vote display

**Contract** — the server sends the raw vote command, the proposer's name and the duration.
The client splits the command into a verb and up to five parameters and rewrites it into
localized text: restart, fast restart, kick, ban, change map, change weather. Anything else is
shown verbatim.

**Invariants** — the kick and ban cases **rejoin the remaining parameters with spaces**,
because a player name can contain spaces and the split does not know that. The map and weather
cases translate their parameter as well, since those are identifiers. This is the client's
only knowledge of the vote vocabulary; the server parses the same strings independently.

**Notes** — the five-parameter cap is arbitrary and a name of more than five words loses its
tail.

## the scoreboard order — `DM_Compare_Players`

**Contract** — spectators sort last; otherwise by frags descending, and among equal frags by
fewer deaths first.

**Invariants** — frags is the derived expression from
[`game_base.cpp`](game_base.cpp.md) — rival kills minus suicides minus team kills — so the
scoreboard penalises both immediately.

## `OnRender`

**Contract** — draws the protection marker over every enemy player who is currently invincible,
but **only when the local player is the one being looked at**, so a spectator watching someone
else does not see it. Each marker uses the team's configured offset, two radii and material.

**Invariants** — the marker exists to tell you not to waste ammunition on someone you cannot
hurt, which is why it is drawn for enemies and not for yourself.

## the remaining hooks

- **`OnSpawn`** — plays the configured spawn particle effect for an actor, and credits a
  weapon appearing in somebody's hands to the telemetry as a purchase.
- **`OnPlayerFlagsChanged`** — refreshes the team panels; for a player who has just become
  permanently dead, closes the local inventory; otherwise pushes the invincibility flag down
  into the actor's condition so damage is actually blocked. This is the one place the flag
  becomes a rule rather than a display.
- **`OnRankChanged`** — reconfigures the buy screen for the new rank, restates costs, and plays
  the rank announcement. Rank zero is silent, because it is where everyone starts.
- **`OnGameRoundStarted`** — closes the buy screen, restores money and rank enforcement, reloads
  the rank-gated defaults and the team preset, clears the last-used preset, and closes the
  inventory.
- **`OnGameMenuRespond_ChangeSkin`** — the server confirmed a skin. Adopt it, close the skin
  screen, prepare the buy screen, and — if the sequence began with a ready press — synthesize
  a jump key press to continue it.
- **`SendPickUpEvent`** — overrides the base to send the *forced* multiplayer ownership event
  rather than the ordinary one, and does **not** apply the base's one-second local touch deny.
  In multiplayer the server arbitrates pickups, so predicting one is wrong.
- **`UpdateMapLocations`** — ensures the local player has a marker on his own map.
- **`GetPlayersPlace`** — sorts a copy of the player table and reports a player's position.
  Called per display, so the sort is repeated; with a few dozen players that is acceptable.
- **`ConvertTime2String`** — hours, minutes and seconds, zero-padded.
- **`PlayParticleEffect`** — creates a one-shot effect at a position and hands it to the
  persistent game object, which owns it until it finishes.

**Notes** — a table of thirty-two English ordinal strings ("1st" … "32th") sits at file scope
and is never referenced; the place is displayed as a bare number. The table also misspells the
fourteenth entry as a second "15th". It is dead.

The minimap contribution is entirely commented out, so deathmatch draws no player markers; what
it would have drawn was every living player in one colour, which is why it was removed —
deathmatch has no allies.
