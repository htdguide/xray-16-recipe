# src/xrGame/DemoInfo.cpp

> The summary block at the head of a recorded multiplayer demo: which map, which mode, the final score, who recorded it, and a per-player scoreboard — written from the live match, read back by the demo browser, and exported to scripts.

**Needs** — [`DemoInfo.h`](DemoInfo.h.md) · [`Level.h`](Level.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_sv_mp.h`](game_sv_mp.h.md) · [`xrCore/stream_reader.h`](../xrCore/stream_reader.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`DemoInfo.h`](DemoInfo.h.md)
**Tier floor** — T2: a fixed field sequence over a stream, plus a derived score

## Purpose

A demo file must be describable without replaying it, so it carries a summary the browser
can read from the front. This file defines that summary, in three directions: build it
from the running match, write it, read it. The on-disk shape is frozen only against
itself — both ends are this codebase — but it is *bounded*, and the bounds are asserted on
read, because the browser reads files it did not necessarily write.

The one piece of real logic is the per-player ranking score, which is computed once when
the demo is recorded and then stored. Computing it at record time rather than at display
time means a demo's scoreboard cannot be re-ranked later by a changed formula, which is
either a bug or a feature depending on whether you think old demos should be stable.

## State

```text
RECORD DemoPlayerInfo
  name       : text
  frags      : int (16-bit, signed)   # kills of players on other teams
  deaths     : int (16-bit, signed)
  artefacts  : int (16-bit)           # artefacts delivered
  spots      : int (16-bit, signed)   # the derived ranking score; see load_from_player
  team       : int (8-bit)            # 0 or 1, or 2 for a spectator
  rank       : int (8-bit)

RECORD DemoInfo
  map_name, map_version, game_type, game_score, author_name : text
  players : list<DemoPlayerInfo>      # owned
```

Bounds, asserted on read and relied on by the loader: each string is at most 256 bytes;
the five header strings together at most 1280; the player count is below the game's
maximum; a player record is at most the string bound plus 80 bytes. These are what let a
reader size a buffer before parsing.

## `demo_player_info::read_from_file` / `write_to_file`

**Contract** — the per-player wire shape, a fixed field sequence with no tags and no
lengths beyond the name's own: name, frags, deaths, artefacts, spots, team, rank. Both
directions must name the fields in exactly this order and at exactly these widths; nothing
versions it.

## `demo_player_info::load_from_player`

**Contract** — snapshots one live player into the record, and computes the ranking score.

```text
FUNCTION load_from_player(player)
  name      = player.name
  frags     = player.kills of rivals
  deaths    = player.deaths
  artefacts = player.artefacts delivered
  rank      = player.rank

  spots = frags - 2 * player.team_kills - player.self_kills + 3 * artefacts

  team = the game mode's own remapping of player.team
  IF team is negative        THEN team = 2       # spectator
  IF the mode is deathmatch AND team is not spectator THEN team = 0
```

**Notes**

- The score formula is the file's only real decision: a team kill costs two frags, a
  suicide costs one, and a delivered artefact is worth three. Those weights are the
  multiplayer scoring policy, compiled in here rather than configured, and the demo
  browser sorts by this number.
- Collapsing every deathmatch participant into team zero exists because the demo format
  always has a team field while deathmatch has no teams; the browser would otherwise
  group players by whatever arbitrary team the match assigned them.

## `demo_info::read_from_file`

**Contract** — reads the five header strings, asserts their combined size against the
bound, reads the player count, asserts it against the maximum, then reads that many player
records. Discards any players already held. Blocks on the stream; a malformed file fails
hard rather than returning an error, which is why the loader
([`DemoInfo_Loader.cpp`](DemoInfo_Loader.cpp.md)) treats an unreadable demo as fatal.

**Invariants** — the size assertions are the only validation. A file that satisfies them
but is otherwise corrupt will produce garbage strings rather than a diagnosis.

## `demo_info::write_to_file`

**Contract** — the mirror: five strings, a count, then the records. The count written is the
actual list size, not the field the reader populates, so writing a summary that was read
and then modified round-trips correctly.

## `demo_info::load_from_game`

**Contract** — snapshots the running match. Takes the map's name and version from the
level, the mode's short name and the current score string from the client-side game mode,
and the recording player's own name as the author — falling back to a literal `unknown`
when the local player has no name. Then snapshots every player in the mode's ordering.

**Notes** — "short" game-type names are used, because the field is displayed in a list.
The author is the *local* player, which means a demo recorded by a spectator is attributed
to the spectator.

## `demo_info::sort_players`

**Contract** — reorders the player list in place by a caller-supplied comparator. The
summary does not choose its own ordering; the loader imposes one (team, then score
descending) immediately after reading, so every consumer sees the same order without
knowing the rule.

## `demo_info::get_player` / the accessors

**Contract** — read-only views of every field. Indexing out of range is a hard failure, not
a `none` — callers are expected to have asked for the count first.

## Script registration

**Contract** — both records are exported to Lua under their own names, with every accessor
and nothing else: a script can read a demo summary and a player's line from it, and cannot
construct or modify either. No constructor is exported, so a script obtains a summary only
from the loader. Names and signatures are frozen by
[conformance criterion 10](../../SYSTEM-REQUIREMENTS.md#6-conformance).

## Could not recover

- A pair of bounded string read/write helpers is defined in comments at the top of the
  file, implementing a length-prefixed string that truncates oversized values and seeks
  past the remainder. The live code uses plain zero-terminated strings instead, so the
  bounds are asserted rather than enforced — a recorded name longer than the bound would
  produce a file the reader refuses. The commented version is the safer design and was
  abandoned for no recorded reason.
- The player-count field read from the file is stored and then ignored in favour of the
  list's own size, so the two can disagree after a modification.
