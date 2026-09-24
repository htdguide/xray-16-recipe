# src/xrGame/character_rank.cpp

> Turns a character's numeric rating into a named rank band, and holds the tables that say how ranks regard each other and what a kill is worth.

**Needs** — [`character_rank.h`](character_rank.h.md) · [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`ini_id_loader.h`](ini_id_loader.h.md) · [`ini_table_loader.h`](ini_table_loader.h.md)
**Used by** — reached through its declarations in [`character_rank.h`](character_rank.h.md); callers name that, not this file.
**Tier floor** — T3: a banding function and two lookup tables

## Purpose

Rank is not stored; it is *derived*. A character carries a rating number, and this file
turns that number into one of a handful of named bands — novice through veteran and up —
using thresholds from configuration. Every rank-dependent decision in the game then works on
the band index rather than the raw number.

The banding is the whole substance. Beyond it, the file owns two global tables: how a
character of one rank regards a character of another, and how much rating a kill of each
rank is worth, which is what closes the loop — killing high-ranked characters raises your
own rating and therefore your own rank.

## State

```text
RECORD RANK_DATA              # one per band, built once from configuration
  id        : text            # the band's authored name
  index     : int             # its position, in ascending threshold order
  threshold : int             # the rating at which this band ENDS

RECORD CHARACTER_RANK
  current_value : int         # the raw rating; the thing that is stored and saved
  current_index : int         # the derived band; always consistent with current_value
```

Shared, one copy for the whole game:

```text
relation_table  : list<list<goodwill>>   # square; [from band][to band]
rank_kill_table : list<list<int>>        # one column: rating awarded for a kill of this band
```

**Invariants**

- `current_index` is a pure function of `current_value` and is recomputed on every write.
  They can never disagree, which is why the value has no independent setter.
- The bands' thresholds must be **ascending** in the authored order. The banding walks them
  in order and takes the first one the value falls under; an out-of-order list silently
  assigns wrong bands with no complaint anywhere.
- The rating that is saved and networked is the raw value. The band is derived on load, so
  changing the thresholds in configuration re-bands every existing character — which is the
  intent: a mod retunes progression without touching saves.
- The relation table is square and indexed by band, not by rating.

## configuration

**Contract** — the band list is the `rating` line of the `game_relations` section, in
ascending order; each band's entry supplies its threshold. The square rank-to-rank goodwill
table is the section `rank_relations`; the per-band kill award is the one-column section
`rank_kill_points`.

## `ValueToIndex`

**Contract** — the banding. Walks the bands in order and returns the first whose threshold
the value is strictly below; a value at or above every threshold lands in the top band.
Pure, static, and the only place the mapping is defined.

```text
FUNCTION value_to_index(rating) -> int
  FOR EACH band IN bands IN ORDER
    IF rating < band.threshold THEN RETURN band.index
  RETURN highest band index          # above every threshold
```

**Notes** — a threshold is the value at which the band *ends*, not the value at which it
begins, and the comparison is strict. The top band's threshold is therefore never used and
can be any number; the shipped data sets it to something large.

## `set` and `id`

**Contract** — `set` records a rating and re-derives the band. `id` gives the current band's
authored name, which is what the interface displays and what scripts compare against.

## `relation`

**Contract** — one cell of the square table: how a character of one rank band regards a
character of another. Static, with an instance form that fills in the reader's own band.
Unlike the community table, there is no writer: rank relations are authored and fixed, and
nothing in the game changes how ranks feel about each other at run time.

## `rank_kill_points`

**Contract** — the rating awarded for killing a character of the given band. The feedback
term of the progression system: the award is a property of the *victim's* band, so killing
upward is worth more than killing downward, and a character's rank rises by taking on people
above them.

## `InitIdToIndex` and `DeleteIdToIndexData`

**Contract** — name the configuration section, the band line and the two tables, and tear
them down. See [`character_community.cpp`](character_community.cpp.md) for the pattern these
three files share.
