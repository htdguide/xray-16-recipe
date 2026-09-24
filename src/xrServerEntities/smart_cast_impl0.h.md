# src/xrServerEntities/smart_cast_impl0.h

> Declaring mode for the cast table: each entry becomes a forward declaration plus one more row in the compile-time table.

**Needs** — [`smart_cast.h`](smart_cast.h.md) · [`smart_cast_impl1.h`](smart_cast_impl1.h.md)
**Used by** — [`smart_cast.h`](smart_cast.h.md) · [`smart_cast_impl1.h`](smart_cast_impl1.h.md)
**Tier floor** — T1: the table is assembled by the compiler itself, from a structure the language has to be coaxed into building.

## Purpose

Defines what a table entry *means* when the table is read to declare casts rather than
define them. An entry names a target type, a source type and a facet method; in this mode it
produces a declaration that such a cast exists, and adds the pair to the accumulating table.

## The table's shape

The table is a list of **groups keyed by source type**, each group holding the source type
followed by every target reachable from it in one hop:

```text
TABLE = list of (source, list<target>)
```

Adding an entry (target, source) is a fold over that structure:

```text
FUNCTION add_to_table(table, source, target)
  IF table has a group whose source is source
    RETURN table with target prepended to that group's targets
  ELSE
    RETURN table with a new group (source, [target]) prepended
```

**Invariants** — the "has a group whose source is source" test matches on **exact type
identity**, not on a base/derived relationship. Two entries whose sources differ only by a
typedef therefore produce two groups, and a later search must handle that.

**Notes** — the table is a *compile-time value*, and the language of the original has no way
to express an accumulating compile-time value except as a chain of named types, each defined
in terms of the previous. That is why every entry in
[`smart_cast.h`](smart_cast.h.md) is followed by a line that re-binds the table's name to
the result. The mechanism is entirely incidental; the structure it builds is not, and a
rebuild produces the same table from a data file or a code generator in a few lines.

The library supplying the compile-time list machinery is included in its abbreviated form,
which cuts build time at the cost of a lower length limit — and the table is well under it.
