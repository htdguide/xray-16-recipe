# src/xrGame/character_reputation.cpp

> Turns a character's numeric reputation into a named band, and holds the table of how those bands regard each other.

**Needs** — [`character_reputation.h`](character_reputation.h.md) · [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`ini_id_loader.h`](ini_id_loader.h.md) · [`ini_table_loader.h`](ini_table_loader.h.md)
**Used by** — reached through its declarations in [`character_reputation.h`](character_reputation.h.md); callers name that, not this file.
**Tier floor** — T3: a banding function and one lookup table

## Purpose

The third axis of how two characters regard each other, after community and rank.
Reputation is *moral standing* rather than skill: it moves on what a character does —
killing neutrals, helping strangers — where rank moves on what they kill. Like rank it is
stored as a number and presented as a band.

Structurally this is the same file as [`character_rank.cpp`](character_rank.cpp.md) with one
table instead of two: the bands are authored with ascending thresholds, a value is banded by
the first threshold it falls under, and a square table says how one band regards another.
The three attitude axes are kept as three separate files precisely so that each can be
retuned in data without touching the others, and so that whatever combines them into one
attitude does so explicitly rather than by accident.

## State

```text
RECORD REPUTATION_DATA        # one per band, built once from configuration
  id        : text
  index     : int
  threshold : int             # the reputation at which this band ENDS

RECORD CHARACTER_REPUTATION
  current_value : int         # the raw reputation; the stored and saved quantity
  current_index : int         # the derived band
```

Shared, one copy for the whole game:

```text
relation_table : list<list<goodwill>>   # square; [from band][to band]
```

**Invariants** — as in [`character_rank.cpp`](character_rank.cpp.md): thresholds ascending;
band always re-derived from the value; the raw value is what persists, so re-tuning the
thresholds re-bands every saved character.

## configuration

**Contract** — the band list is the `reputation` line of the `game_relations` section; the
square table is the section `reputation_relations`.

## `ValueToIndex`

**Contract** — the banding: the first band whose threshold the value is strictly below,
falling through to the top band. Pure and static.

```text
FUNCTION value_to_index(reputation) -> int
  FOR EACH band IN bands IN ORDER
    IF reputation < band.threshold THEN RETURN band.index
  RETURN highest band index
```

## `set`, `id`, `relation`

**Contract** — `set` records a reputation and re-derives its band; `id` names the band;
`relation` reads one cell of the square table, with an instance form supplying the reader's
own band. The table is read-only: authored data decides how reputations regard each other,
and nothing changes it at run time.

## `InitIdToIndex` and `DeleteIdToIndexData`

**Contract** — name the configuration section, the band line and the table, and tear them
down.
