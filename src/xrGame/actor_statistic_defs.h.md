# src/xrGame/actor_statistic_defs.h

> The shape of the player's end-of-game scorecard: named sections, each a list of named tallies, persisted with the save.

**Needs** — [`alife_abstract_registry.h`](alife_abstract_registry.h.md) · [`actor_statistic_mgr.cpp`](actor_statistic_mgr.cpp.md)
**Used by** — [`actor_statistic_mgr.cpp`](actor_statistic_mgr.cpp.md) · [`actor_statistic_mgr.h`](actor_statistic_mgr.h.md) · [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md)
**Tier floor** — T2: serialized records with a version-gated field

## Purpose

The game keeps a running scorecard of the player's deeds — stalkers killed, monsters
killed, tasks completed, artefacts found, reputation earned — and shows it on the statistics
screen and at the end of the game. This declares its shape. The shape is deliberately
*open*: sections and tallies are named by string and created on first use, so new score
categories are added by whoever scores them and never by editing an enumeration.

Behaviour is in [`actor_statistic_mgr.cpp`](actor_statistic_mgr.cpp.md).

## State

```text
RECORD Tally
  key       : text     # what is being counted, e.g. a creature class
  count     : int (32-bit, signed)
  points    : int (32-bit, signed)
  text_value: text     # when non-empty, this tally is a DISPLAYED STRING, not a score

RECORD Section
  key   : text            # the score category, e.g. "stalkerkills"
  data  : list<Tally>     # created on demand; order is insertion order and is the save order

REGISTRY actor_statistics : list<Section>    # one per player; alife-registered, so it
                                             # survives level changes and rides the save
```

Invariants:

- A tally is *either* numeric or textual. A non-empty text value marks the tally as
  unscorable, and a section containing one has no numeric total at all — see the
  aggregation rule in the implementation twin. The union is expressed by convention, not
  by a tag.
- Both collections are searched linearly by key and appended to on miss. They hold a few
  dozen entries at most and are touched on gameplay events, not per frame.
- The registry is keyed by the player's entity identifier even though there is only ever
  one player in single player; the key exists because the registry type is shared.

## Serialization

**Contract** — each record writes its own fields in a fixed order. A section writes its
tallies *before* its key; a tally writes key, count, points, then text value. Reading is
the mirror, with two version gates.

**Invariants** — the format has three generations and the reader handles all of them by
consulting the save's alife version:

```text
FUNCTION load_tally(reader, save_version)
  key    = read_text(reader)
  count  = read_int(reader)
  points = read_int(reader)
  IF save_version > 2 THEN text_value = read_text(reader)
  # older saves have no text tallies at all

FUNCTION load_section(reader, save_version)
  data = read_list_of_tallies(reader)
  IF save_version == 2 THEN
    numeric_key = read_int(reader)
    key = the name that numeric code used to mean   # see the fixed table below
    discard read_int(reader)                        # a precomputed total, no longer stored
  ELSE
    key = read_text(reader)
```

The frozen translation for version-2 saves, which is the only place these numbers exist:

```text
100 -> "total"          1 -> "stalkerkills"     2 -> "monsterkills"
  3 -> "quests"         4 -> "artefacts"        5 -> "reputation"
  0 -> "foo"
```

**Notes**

- The design changed from a closed enumeration of score categories to open string keys,
  and the reader carries the old table so that saves from before the change still load.
  The `foo` entry is a placeholder that was in the shipped enumeration; a save can
  legitimately contain it.
- The old format also stored each section's total, which is now always recomputed. Storing
  a derived value is what made the change necessary: the totalling rule changed and the
  stored values were wrong.
- A named token table for the score categories is declared here and defined nowhere in
  this fork; the console variable that would have used it is gone. A rebuild should drop
  the declaration.
