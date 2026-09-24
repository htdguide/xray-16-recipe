# src/xrGame/game_cl_base_weapon_usage_statistic_save.cpp

> Writes the match telemetry out: a binary report file named for the level, the mode and the clock, and a parallel text report in the configuration format.

**Needs** — [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`Level.h`](Level.h.md)
**Used by** — [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md)
**Tier floor** — T1: writes fixed-width records straight from memory to a file

## Purpose

The persistence half of the telemetry declared in
[`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md). It exists
separately because it is the only part that touches the disk, and because there are **two
independent output formats** that share nothing but the data: a compact binary report, and a
text report in the engine's own configuration format for a tool to read.

Both are server-only and both refuse to write when collection was off or nothing was
collected.

## State

`Stateless.` The collector's state is declared in the header.

## the binary report

**Contract** — a four-byte magic tag, a four-byte version, then the match totals and every
player.

```text
FILE weapon_usage_report
  magic    : 4 bytes, the characters of a fixed tag
  version  : int (32-bit)      # 2; version 1 lacked bone names
  totals   : three arrays of three 32-bit ints   # alive time, money, respawns per team index
  players  : count, then that many player records

RECORD player_record
  name          : null-terminated text
  total_shots   : int
  alive_time    : int[3]
  money_round   : int[3]
  respawns      : int[3]
  weapons       : count, then that many weapon records

RECORD weapon_record
  name, inventory_name : null-terminated text
  bought, rounds_fired, bullets_fired, hits, kills : int
  purchase_histogram   : 3 x 34 ints
  hits                 : count of COMPLETED hits, then that many hit records

RECORD hit_record
  from, to    : three 32-bit floats each
  bone_id     : int (16-bit)
  deadly      : one byte
  target_name : null-terminated text
  bone_name   : null-terminated text
```

**Invariants** — only **completed** hits are written, and the count is computed by a first
pass over the list before the second pass writes them. The two passes must apply the same
predicate.

The file is written with the host's byte order and the host's float representation, directly
from memory. It is a memory image, readable only by a tool built with the same assumptions —
which for this engine means little-endian and 32-bit floats (see the platform assumptions).

The version tag is 2, and the difference from version 1 is that hit records carry a bone
name. That is the only compatibility break recorded.

**Notes** — the hit record does **not** carry the repeat count, although the record in memory
has one. A run of twenty identical hits collapsed into one row is written as one hit and the
multiplicity is lost. The text report has the same field missing but compensates, as below.

The report writes hits but not the two kill-kind counters, and the profile identifier and
account digest are absent entirely. The binary format is therefore strictly poorer than the
text one, and a rebuild that keeps only one should keep the text.

## the report file name

**Contract** — composed from the level's name, a short mode tag, and the local date and time,
with a fixed extension, placed under the logs root. An unrecognised mode aborts the write
entirely.

**Invariants** — the name is the only index: there is no manifest, and a tool reading reports
learns the level, mode and time from the file name alone. So the name's shape is part of the
format.

**Notes** — the date is formatted as month, day, year on one platform and as the platform's
default timestamp string on the others, so the two are not interchangeable and the
non-Windows one embeds a trailing newline in the file name. A rebuild should use one
sortable, platform-independent timestamp.

## the text report

**Contract** — the same data written into the engine's configuration format: one section for
the match totals, one section per player, one section per weapon, and hits as prefixed keys
within their weapon's section.

**Invariants** — three differences from the binary report are decisions, not omissions.

- **Only players with an account digest are written**, and the count written is the count of
  those, not of all players. A player the server never received a digest for — one who never
  answered a pull — is dropped from the report entirely. The report is about accounts, not
  about connections.
- **Alive times are converted from milliseconds to seconds** on the way out, in the totals and
  per player. The binary report writes milliseconds. Two reports of the same match therefore
  carry the same quantity in different units.
- **Repeated hits are expanded.** Where a hit row stands for a run of identical hits, the text
  report emits that many consecutive hit entries with distinct indices and identical content.
  This is how the multiplicity that the binary format loses survives — at the cost of the row
  appearing several times.

The player index in the section name is the index among *written* players, so it is dense;
the weapon index is the index in the player's list, so it is dense too. Both are needed
because the format has no arrays.

**Notes** — the expansion loop advances two counters, one over rows and one within a row, and
resets the within-row counter when it skips an incomplete row. A row with a repeat count of
zero would loop forever; the append path never produces one, since a row starts at one.

Each hit's key names are built by concatenating a per-hit prefix with a field name, which is
how a flat key-value format carries a list.

The text report includes the two kill-kind counters and the account identity that the binary
report omits, and omits the purchase histogram that the binary report includes. Neither
format is complete.
