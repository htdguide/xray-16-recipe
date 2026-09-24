# src/xrGame/character_community.cpp

> The faction a character belongs to, and the shared tables that say how any two factions feel about each other.

**Needs** — [`character_community.h`](character_community.h.md) · [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`ini_id_loader.h`](ini_id_loader.h.md) · [`ini_table_loader.h`](ini_table_loader.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two square lookup tables loaded from configuration

## Purpose

Every character belongs to a *community* — a faction — and almost every decision about
whether two characters shoot each other starts from what their two communities think of one
another. This file holds a character's community as a value, and owns the two global tables
that make that value mean something.

The design decision worth carrying forward is the **name-to-index collapse**. Communities
are authored as strings, but the relation between every pair of them is a square table read
every time anyone evaluates an attitude, and string-keyed lookups in that path are
unaffordable. So the community list is read once at startup, each name is assigned a
position, and everything downstream — the relation table, the sympathy table, a character's
stored community — is that position. A string appears only at the boundaries: reading
configuration, and answering a script.

## State

```text
RECORD COMMUNITY_DATA         # one per community, built once from configuration
  id    : text                # the authored name
  index : int                 # its position; the key everything else uses
  team  : int (8-bit)         # the multiplayer team this community maps to

RECORD CHARACTER_COMMUNITY
  current_index : int         # NO_COMMUNITY_INDEX when unset
```

Shared, one copy for the whole game:

```text
relation_table  : list<list<goodwill>>   # square; [from][to]
sympathy_table  : list<list<real>>       # one column
```

**Invariants**

- The relation table is square and its side equals the number of communities. It is **not**
  symmetric: what A thinks of B is a separate entry from what B thinks of A, which is how
  the data expresses one-sided hostility.
- The sympathy table has one row per community and exactly one column. Declaring it as a
  table with a fixed column count rather than a list is what lets it share the loader with
  the relation table; it is a vector wearing a table's shape.
- A community's index is only valid for the run in which the list was loaded. Indices are
  positions in the authored list, so inserting a community in the middle renumbers
  everything after it. Nothing that outlives the process — a save, a network message — may
  store an index; it must store the name.
- The "no community" index is a distinct value outside the table's range and must be checked
  before any table access. A character with no community is legitimate (a neutral entity,
  an item's nominal owner) and indexing on it is out of bounds.

## configuration

**Contract** — the community list is one comma-separated line, `communities`, in the
`game_relations` section; the order of that line *is* the index assignment. Each community's
own entry supplies its team number. The two tables are separate sections named
`communities_relations` (square) and `communities_sympathy` (one column).

## `set` and `id`

**Contract** — `set` takes an authored name and resolves it to an index, optionally without
asserting on an unknown name; the unchecked form exists because saves and mods can name a
community the current data set does not define, and hard-failing there makes a save
unloadable. `id` converts back. A second `set` takes an index directly for the paths that
already have one.

## `team`

**Contract** — the multiplayer team number this community maps to. Community and team are
two vocabularies for the same thing — the single-player world groups characters into
factions, the multiplayer modes group them into numbered teams — and this is the bridge, so
that one set of relation rules can serve both.

## `relation` and `set_relation`

**Contract** — read and write one cell of the square table: how community *from* regards
community *to*, as a goodwill value. Both forms are static (the table is global); the
instance form fills in *from* with the character's own community. Writing is how the game
changes faction politics at run time — a script can make a faction hostile to the player
permanently, and the change is global and immediate, affecting every member at once.

**Invariants** — a goodwill may not be written as the "no goodwill" sentinel. That value
means *absent*, and an absent cell in a square table is a data error, not a relationship.

**Notes** — the table is mutable global state with no save hook of its own. Whatever
persists faction relations across a save has to walk this table and write it; this file does
not.

## `sympathy`

**Contract** — one number per community: how strongly its members react to something
happening to a fellow member. It is the coefficient by which a goodwill change aimed at one
character propagates to the rest of his faction, which is what makes shooting one member of
a faction turn the faction against you rather than only the victim.

## `InitIdToIndex` and `DeleteIdToIndexData`

**Contract** — name the configuration section, the list line and the two tables, and tear
them down. They exist because the index machinery is generic and has to be told, once per
kind of identifier, where its data lives.

**Notes** — this file, [`character_rank.cpp`](character_rank.cpp.md) and
[`character_reputation.cpp`](character_reputation.cpp.md) are three instances of one
pattern: an authored list of names collapsed to indices, plus a square relation table
between them. A rebuild should write the pattern once and instantiate it three times, which
is what the original does too; the three files differ only in what a value *is* (a
membership here, a band of a number in the other two) and in what extra per-item column they
carry.
