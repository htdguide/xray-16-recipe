# src/xrGame/game_base.cpp

> The per-player scoreboard record and its wire format, and the game clock: two independently rescalable in-world times derived from the server's real time.

**Needs** — [`game_base.h`](game_base.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`player_account.h`](player_account.h.md) · [`gametype_chooser.h`](../xrServerEntities/gametype_chooser.h.md) · [`xrScriptEngine/script_engine.hpp`](../xrScriptEngine/script_engine.hpp.md)
**Used by** — [`game_base.h`](game_base.h.md)
**Tier floor** — T2: a wire format with fixed field widths, over a monotonic clock

## Purpose

Two unrelated things share this file because both are "state of the match that is neither
server-only nor client-only": the player record that the server owns and every client mirrors,
and the clock that turns real elapsed time into in-world time. Splitting them would be
reasonable; they are together because both are members of the game-rules base class's world.

## State

```text
RECORD PlayerState
  team                : int (8-bit)      # 0, 1, or the spectator slot
  rival_kills         : int (16-bit)     # kills of the other team
  self_kills          : int (16-bit)     # suicides
  team_kills          : int (16-bit)
  kills_in_row_curr   : int (16-bit)     # current streak; server-side only
  kills_in_row_max    : int (16-bit)     # best streak this match; server-side only
  deaths              : int (16-bit)
  money_for_round     : int (32-bit, signed)
  experience_real     : real             # server-side only
  experience_new      : real             # server-side only
  rank                : int (8-bit)
  af_count            : int (8-bit)      # objectives delivered, in the artefact modes
  flags               : int (16-bit)     # the bit set below
  ping                : int (16-bit)
  game_id             : int (16-bit)     # the entity identifier of this player's body
  lasthitter          : int (16-bit)     # who last damaged this player
  lasthitweapon       : int (16-bit)
  skin                : int (8-bit, signed)
  respawn_time        : int              # server-side only
  death_time          : int              # see the wire-format note
  money_delta         : int (16-bit, signed)
  vote_agreed         : int (8-bit)      # 0 no, 1 yes, 2 not yet voted
  old_ids             : queue<int>       # the last ten bodies; at most ten
  money_added         : int (32-bit, signed)
  bonus_money         : list<BonusLine>
  pay_for_spawn       : bool
  online_time         : int
  account             : PlayerAccount    # name and persistent profile
  player_ip           : text             # server-side only
  player_digest       : text             # server-side only
  items               : list<int>        # entity identifiers of the loadout
  spawn_points        : list<int>        # which respawn points this player may use
  last_spawn_point    : int (16-bit, signed)   # -1 when none
  last_buy_account    : int (32-bit, signed)
  clear_run           : bool
```

**Invariants**

- **Frags are derived, never stored**: a player's score is rival kills minus suicides minus
  team kills. Every scoreboard reads that expression; nothing writes a frag count. A rebuild
  that caches it will drift.
- The `old_ids` queue holds **at most ten** previous body identifiers, oldest dropped. It
  exists because a death message or a hit report can arrive after the player has already
  respawned into a new body, and the receiver must still attribute it to the right player.
  Ten is a depth, not a semantic: it is more respawns than any in-flight message survives.
- The three flags a rebuild must get right: *local* marks the record describing this
  client's own player; *skip* marks a record that carries no account and must not be
  imported or displayed — the dedicated server sets it on its own phantom player, since a
  dedicated server has no profile to load; *spectator*, *ready*, *invincible*, *on base* and
  *very very dead* are mode-visible states. Bits from the fifth upward are reserved for
  scripts, which is why the flag word is exported to Lua as a raw integer with test, set and
  reset calls rather than as named properties.
- Half the record never crosses the wire. Streaks, experience, respawn time, the item list,
  the spawn-point list, the address and the digest are server-side; the export below is the
  authority on what a client may know.

## `PlayerState` construction

**Contract** — clears every counter, then takes one of three paths depending on where the
record comes from. Given an account packet — that is, a remote player the server has just
told us about — it imports it. Given nothing, on a dedicated server it marks itself as a
record to skip; given nothing on a client, it loads the local player's persistent profile
from disk.

**Invariants** — the "given nothing" path is for the *local* player only. Constructing a
record with no account for a remote player silently adopts the local profile, which is why
the two paths are distinguished by the presence of the packet rather than by a flag.

The vote field initialises to 2, meaning "has not voted", so that a vote in progress can
distinguish abstention from a no.

## `clear`

**Contract** — resets everything that belongs to one round: all six counters, the last
hitter and weapon, experience, rank, objective count, the loadout, the spawn-point
assignment, the bonus-money breakdown, the address and digest, and the body-identifier
history. Leaves team, skin, name, ping, money for the round and the persistent account
alone — those survive a round change.

**Invariants** — what `clear` does *not* touch is the definition of "what persists across a
round" and is the only place it is written down.

## `net_Export` / `net_Import`

**Contract** — the wire format of a player record, one byte per field in a fixed order,
prefixed by a flag saying whether the persistent account follows. A partial update carries
the volatile fields only; a full update appends the account.

```text
FUNCTION export(packet, full)
  packet.write_int8(full ? 1 : 0)
  write in this exact order:
    team, rival_kills, self_kills, team_kills, deaths,
    money_for_round, rank, af_count, flags, ping,
    game_id, skin, vote_agreed,
    (now - death_time)                    # note: an age, not a timestamp
  IF full THEN account.export(packet)
```

**Invariants** — the field order *is* the format; import reads the same order and must not
be reordered independently.

The death time is exported as **elapsed time since death** but imported straight into the
field that held an absolute timestamp. The receiver therefore stores an age where the sender
stored an instant, and any client-side arithmetic against its own clock is wrong. Exporting
an age is the right decision — the two machines' clocks do not agree — but the receiver must
convert it back by subtracting from its own now, and does not. Reproduce the transmission,
fix the reception.

## `skip_Import`

**Contract** — reads and discards exactly the same field sequence, including the account
when the full flag is set. It exists because the packet is a stream: a record that cannot be
applied still has to be stepped over so the records behind it decode.

**Invariants** — this is a *third* copy of the field order. All three must change together;
a rebuild should generate all three from one description.

## `SetGameID`

**Contract** — records the current body identifier in the history (dropping the oldest when
ten are held) and adopts the new one.

## `HasOldID`

**Contract** — whether the given identifier was one of this player's recent bodies. Used to
attribute a late message to the right player.

## the game clock

**Contract** — in-world time is not counted; it is *computed* from the server's real time
through a linear map, so that no accumulation error builds up and so that a client and a
server that agree on real time agree on world time exactly.

```text
RECORD Clock
  origin_real  : int    # server real time when the rate last changed
  origin_game  : int    # in-world time at that instant
  factor       : real   # in-world milliseconds per real millisecond

FUNCTION now() -> game_time
  RETURN origin_game + factor * (server_real_time_now - origin_real)

FUNCTION set_factor(new_factor)
  origin_game = now()            # rebase first, or the change rewrites history
  origin_real = server_real_time_now
  factor      = new_factor

FUNCTION set_factor(at_game_time, new_factor)
  origin_game = at_game_time     # jump the world clock and rescale in one step
  origin_real = server_real_time_now
  factor      = new_factor
```

**Invariants** — rebasing before changing the rate is the whole correctness argument: the
new rate must apply only from now forward, never retroactively to the elapsed interval.

Two independent instances of this clock exist per match with the same structure: the **game
clock**, which alife, mission timers and the save file read, and the **environment clock**,
which only the weather and sky system reads. They are initialised to the same origin and
rate and diverge only when something rescales one of them.

**Notes** — the defaults are noon (twelve hours past midnight, in milliseconds) for both
clocks and a rate of ten: in-world time runs ten times faster than real time, so a full
in-world day takes a little under two and a half real hours. Those three values are module
globals the level loader overwrites from the save or the level's data; they are the values
used when nothing else says otherwise.

The clock reads the *asynchronous* server time — a value sampled without taking the network
lock — because it is read from the render thread as well as the simulation thread and must
never block.

## `switch_Phase`

**Contract** — notifies the mode of the old and new phase *before* adopting the new one,
then stamps the phase's start time from the server clock.

**Invariants** — the notification runs before the field changes, so a mode's handler sees
the outgoing phase as current. Every mode's handler is written against that ordering.

## `getCLASS_ID`

**Contract** — maps a mode's name and a side (server or client) to the class identifier of
the class that implements it. Five modes, each with two halves: single player, deathmatch,
team deathmatch, artefact hunt, capture the artefact. An unknown name yields a blank
identifier, which the factory then fails to resolve.

**Notes** — the identifiers are eight-character tags padded with spaces, because class
identifiers are a fixed-width tag in the spawn data (see the glossary). The padding is part
of the value.

A commented-out earlier version resolved the mapping by calling a Lua function named in a
configuration file, so that a mod could add a game mode without touching the engine. It was
replaced by this fixed table. A rebuild wanting mod-defined modes should restore the
indirection, not the table.
