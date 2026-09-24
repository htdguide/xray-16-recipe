# src/xrGame/ini_id_loader.h

> Turns a configuration line of names into a dense index space, so that the names can be used as array subscripts everywhere else.

**Needs** — [Seam: Configuration format](../xrCore/README.md)
**Used by** — [`character_community.cpp`](character_community.cpp.md) · [`character_community.h`](character_community.h.md) · [`character_rank.cpp`](character_rank.cpp.md) · [`character_rank.h`](character_rank.h.md) · [`character_reputation.cpp`](character_reputation.cpp.md) · [`character_reputation.h`](character_reputation.h.md) · [`ini_table_loader.h`](ini_table_loader.h.md) · [`monster_community.cpp`](monster_community.cpp.md) · [`monster_community.h`](monster_community.h.md)
**Tier floor** — T2: a name-to-index registry built once from configuration

## Purpose

Several of the game's concepts are open sets named in configuration — factions, reputation
levels, damage kinds — and each of them is used as an index into a table that is also
configuration. A faction's name is what the data says; a faction's *number* is what the
relation matrix is indexed by. This file is the bridge: it reads one line of names, assigns
each the next integer, and answers both directions.

The assignment is **positional**, which is the decision everything downstream rests on: the
index of a name is its position in that line. Reordering the line silently renumbers
everything and invalidates every table indexed by it, and every save file that stored an
index.

## State

```text
RECORD Entry
  index : int      # assigned by position, starting at zero
  id    : text     # the name as it appears in configuration
  ...              # zero or one extra field, per instantiation

# per instantiation, all shared:
entries      : optional<list<Entry>>   # in index order; index == position
section_name : text                    # where the name line lives
line_name    : text
```

**Invariants** — the registry is **per instantiation and shared**: there is one entry list per
(entry type, identifier type, index type) combination, not one per object. A rebuild will
express this as a module-level table rather than as a type-level one; what must survive is
that all users of a given concept see one numbering.

The list is never reordered or compacted, so an index is stable for the lifetime of the
process.

## `InitInternal`

**Contract** — builds the registry once. Asks the initialization policy for the section and
line names, reads that line, splits it into items, and appends one entry per item with the
next index.

```text
FUNCTION initialize()
  REQUIRE the registry is empty        # initializing twice would renumber everything
  policy.set_section_and_line()
  line = configuration.read(section_name, line_name)
  count = number of items in the line
  FOR k FROM 0 WHILE k < count
    name = item k of the line
    IF this instantiation carries an extra field THEN
      extra = item (k+1) of the line ; k += 1     # the extras are INTERLEAVED with the
                                                  # names, not in a separate line
      append Entry(next_index, name, extra)
    ELSE
      append Entry(next_index, name)
```

**Invariants** — when a registry carries an extra field, the line alternates name, extra,
name, extra. An odd item count silently drops or mis-pairs the last entry; nothing checks.

**Notes** — each name is lowercased into a temporary that is then **never used** — the entry is
built from the original, case preserved. The lowercasing is dead, and the source says so. A
rebuild must decide deliberately whether lookup is case-sensitive; as shipped it is, because
the comparison is against the unmodified name.

## `GetById`

**Contract** — the entry for a name, by linear scan. Missing is fatal unless the caller passes
the suppression flag, in which case it answers nothing.

**Notes** — a linear scan over a list of a few dozen, called rarely (at load, when parsing a
script's faction name), so the absence of a map is not a defect. A rebuild that has a map for
free should use one.

## `GetByIndex`

**Contract** — the entry at an index, bounds-checked. Out of range is fatal unless suppressed,
and the failure names the section and line the registry came from — which is the only useful
thing to say, since the fault is almost always that configuration and code disagree about the
name set.

## `IdToIndex` and `IndexToId`

**Contract** — the two conversions, each taking a default to return when the lookup fails and
the suppression flag. The defaults are what lets a caller treat an unknown name as "none"
rather than as an error; the default index is all-ones.

## `GetMaxIndex`

**Contract** — the highest assigned index, one less than the count.

**Invariants** — used as the dimension of every table indexed by this registry; see
[`ini_table_loader.h`](ini_table_loader.h.md). The registry must therefore be initialized
before any such table is built.

## `DeleteIdToIndexData`

**Contract** — releases the registry. Called at shutdown. After it, every index in flight is
meaningless, so nothing indexed by this registry may outlive it.
