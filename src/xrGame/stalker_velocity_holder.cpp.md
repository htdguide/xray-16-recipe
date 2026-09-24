# src/xrGame/stalker_velocity_holder.cpp

> One speed table per configuration section, shared by every stalker that uses it.

**Needs** — [`stalker_velocity_holder.h`](stalker_velocity_holder.h.md) · [`stalker_velocity_collection.h`](stalker_velocity_collection.h.md)
**Used by** — [`stalker_velocity_holder.h`](stalker_velocity_holder.h.md)
**Tier floor** — T2: a keyed cache of loaded tables.

## Purpose

A few hundred stalkers draw their movement speeds from a few dozen configuration sections.
Loading the nineteen-value table per creature would mean parsing the same section hundreds
of times at level load and holding hundreds of identical copies. This is the interning
layer: ask for a section, get the one table for it, loading it the first time.

## State

```text
RECORD StalkerVelocityHolder
  tables : map<text, StalkerVelocityCollection>   # section name -> the loaded table
```

**Invariants** — the holder owns every table it has created, and a table is never evicted.
Lifetimes are bounded by the process, not by the level, so the tables survive a level change
and a second visit to the same level costs nothing. Tables are immutable once loaded, which
is what makes sharing them across creatures safe.

## `collection(section)`

**Contract** — the table for a section, loading it on first request. Blocks on the
configuration layer for a section not yet seen; returns immediately afterwards. The returned
table is shared and must not be modified. Not safe against concurrent first requests for the
same section — creature construction happens on one thread and the code relies on it.

```text
FUNCTION collection(section) -> StalkerVelocityCollection
  IF tables HAS section
    RETURN tables[section]
  table := load_velocity_collection(section)
  tables[section] := table
  RETURN table
```

**Notes** — the map is a sorted flat array rather than a hash table. With a few dozen
entries, looked up by an interned string whose comparison is a pointer comparison, the
linear locality wins; and the container choice is visible in the shape of the code rather
than in the contract, so a rebuild may use whatever its language makes cheap.

## Teardown

**Contract** — destroying the holder destroys every table it loaded. There is no way to
release a single one.
