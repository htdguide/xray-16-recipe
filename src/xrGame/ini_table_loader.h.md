# src/xrGame/ini_table_loader.h

> Loads a configuration section into a two-dimensional table whose rows are addressed by a name registry rather than by position.

**Needs** — [`ini_id_loader.h`](ini_id_loader.h.md) · [Seam: Configuration format](../xrCore/README.md)
**Used by** — [`character_community.cpp`](character_community.cpp.md) · [`character_community.h`](character_community.h.md) · [`character_rank.cpp`](character_rank.cpp.md) · [`character_rank.h`](character_rank.h.md) · [`character_reputation.cpp`](character_reputation.cpp.md) · [`character_reputation.h`](character_reputation.h.md) · [`monster_community.cpp`](monster_community.cpp.md) · [`monster_community.h`](monster_community.h.md)
**Tier floor** — T2: a table built once from configuration

## Purpose

The relation between two factions, the reward for a reputation level, the compatibility of
two item kinds — all are square matrices authored as configuration, one line per row, with
the row named rather than numbered. This file builds such a table.

Two decisions make it more than a parser. First, **rows are placed by the registry's index,
not by the order they appear in the file**, so an author may reorder the section freely.
Second, the table's dimension comes from the registry rather than from the file, so a section
that does not cover the full name set is an error rather than a short table.

## State

```text
RECORD Table
  rows        : optional<list<list<Item>>>   # built lazily on first access
  section     : text                         # where the rows are authored
  width       : int = -1                     # -1 means "square: width equals height"
```

**Invariants** — the table is built lazily and cached. The registry it is indexed by must be
initialized first, because the height is taken from the registry's index count.

A width of minus one means square. That default exists because most of these tables are
relations of a set with itself, and stating the dimension twice invites them to disagree.

## `table`

**Contract** — returns the table, building it on first call.

```text
FUNCTION table() -> rows
  IF already built THEN RETURN it

  height = registry.max_index + 1
  width  = (configured width = -1) ? height : configured width
  rows.resize(height)

  section = configuration.read_section(section_name)
  REQUIRE section has exactly `height` lines      # every name must have a row

  FOR EACH (key, value) IN section
    row = registry.index_of(key)
    FAIL IF row is unknown WITH "wrong name in section"
    rows[row].resize(width)
    FOR j FROM 0 TO width-1
      rows[row][j] = convert(item j of value)
  RETURN rows
```

**Invariants**:

- **The line count must equal the registry's size exactly.** Not at least — exactly. A missing
  row would leave a row of the table unsized and reading it would be undefined; an extra row
  means the section and the registry disagree and one of them is wrong.
- **Each row is placed at its name's registry index**, so the file's order is irrelevant and a
  row for a name the registry does not know is fatal, naming both the name and the section.
- **Only rows that appear are sized.** Combined with the exact-count requirement this is
  sufficient, but it means the two checks are load-bearing together: relax the count check and
  unsized rows become reachable.
- The number of values read per row is the table's width; a row with fewer values reads past
  its end and a row with more is silently truncated. Nothing checks.

## `set_table_params`

**Contract** — names the section and optionally fixes the width. Must be called before the
first access, since the table is built once and cached.

## `convert`

**Contract** — parses one item into the table's element type. Two are supported, whole numbers
and real numbers; anything else is rejected when the table is instantiated rather than at run
time.

**Notes** — parsing is by the permissive numeric conversion, which yields zero for anything
unparseable rather than failing. A typo in a relation matrix therefore reads as a zero
relation, silently. A rebuild should parse strictly; the shipped data is clean, so doing so
changes nothing except the diagnostics.

## `clear`

**Contract** — discards the built table. The next access rebuilds it from configuration, which
is how a configuration reload takes effect.

## The table index parameter

**Notes** — instantiations carry a numeric tag whose only purpose is to make two tables with
the same element type and the same registry distinct types. A rebuild that keys tables by
value rather than by type has no need of it.
