# src/xrGame/monster_community.cpp

> Which species-group a creature belongs to, and the one square table that answers how any two groups feel about each other.

**Needs** — [`monster_community.h`](monster_community.h.md) · [`ini_id_loader.h`](ini_id_loader.h.md) · [`ini_table_loader.h`](ini_table_loader.h.md)
**Used by** — reached through its declarations in [`monster_community.h`](monster_community.h.md); callers name that, not this file.
**Tier floor** — T3: two configuration-loaded tables and an index

## Purpose

The non-human half of the faction system. It is small because all the machinery — parsing a
name list into a dense index space, parsing a square matrix of numbers — is shared with the
human factions and lives in the two loaders. What is here is the three configuration names
this type uses and the two-index lookup.

## State

```text
RECORD MonsterCommunityData       # one row of the process-wide community list
  id    : text                    # authored name, interned
  index : int                     # position in the authored list; the dense key
  team  : int (8-bit)             # which side this group fights on

RECORD MonsterCommunity           # a creature's membership
  current_index : int             # the sentinel until set

# process-wide, shared by every creature:
  community list  : list<MonsterCommunityData>
  relation table  : list<list<int>>     # square, indexed [from][to]
```

**Invariants**
- The relation table is square and its side equals the number of communities. Both come from
  the same configuration section and the loader enforces the agreement.
- An index is valid only in the range of the community list. The lookup asserts both indices
  in development builds and reads out of bounds without one — an unset membership reaching
  the lookup is a caller error, not a condition to handle.
- A membership starts at the "no community" sentinel and must be set before any attitude
  question is asked.
- The attitude table is **not symmetric**. Group A may be hostile to B while B is
  indifferent to A, and the shipped data uses that: predators attack prey that has no
  particular opinion in return.

## Configuration

Three names are read, and they are the frozen part of this file:

- the section `monster_communities`, which carries
- the key `communities` — the authored list of group names, whose *order* defines the index
  space and therefore the row and column order of the table;
- the table `monster_relations` — the square matrix of attitudes.

**Notes** — the index space is positional. Inserting a community into the middle of the
authored list shifts every index after it and silently rotates the relation table's meaning.
Anything that persists a community index rather than a name is therefore tied to the shipped
data's ordering. This is the same hazard the human faction table carries.

## `set` / `id` / `index` / `team`

**Contract** — `set` takes either the authored name — resolved through the shared
name-to-index map — or an index directly. `id` and `index` report the membership in either
form. `team` reads the group's team number out of the process-wide list.

**Notes** — `team` indexes the community list with the current index and will read out of
bounds on an unset membership, with no check even in development builds. Every caller sets
the community first, from the creature's configuration section, so it does not happen; the
absence of the guard is an oversight rather than a decision.

## `relation`

**Contract** — the attitude from one community to another, as an integer. Two forms: a free
one taking both indices, and one relative to this membership. Pure; no side effects.

```text
FUNCTION relation(from, to) -> int
  RETURN relation_table[from][to]
```

**Notes** — the number's *scale* is not defined here. It is a signed quantity that the
creature brains compare against thresholds of their own, and the shipped data uses a small
range around zero with negative meaning hostile. Nothing in this file constrains it, which
means a rebuild must take the scale from the consumers, not from the table.

## `InitIdToIndex` / `DeleteIdToIndexData`

**Contract** — `InitIdToIndex` names the configuration section, the key holding the community
list and the table holding the attitudes; the shared loader does the rest. It is the hook the
loader calls back into, which is how one loader serves both this type and the human faction
type with different data. `DeleteIdToIndexData` releases the attitude table and then the
shared name-to-index data, in that order — the table is sized from the name list and must go
first.

**Notes** — both tables are process-wide and loaded once. They are not per-level and not
per-save: creature attitudes are a property of the game, not of the world's state. A rebuild
that wants script-mutable creature relations has to move them, and the shipped scripts do not
expect to be able to.
