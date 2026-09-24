# src/xrGame/smart_cover_storage.cpp

> Caches parsed smart-cover descriptions so that the same authored cover table is read from configuration once, and frees them only after a grace period.

**Needs** — [`smart_cover_storage.h`](smart_cover_storage.h.md) · [`smart_cover_description.h`](smart_cover_description.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`smart_cover_storage.h`](smart_cover_storage.h.md)
**Tier floor** — T2: sharing and lifetime policy; nothing device- or layout-facing

## Purpose

A *smart cover* object in a level names a table of authored data — its loopholes, their
fields of view, and the transition graph of animations between them. Parsing that table
means calling into the script virtual machine and building a graph, which is far too
expensive to redo per cover instance. This module is the one place that decides how long
a parsed description outlives the last cover that used it.

The interesting decision is not the cache but the **deferred free**. Descriptions are
reference-counted by their users; when the count reaches zero the description is *not*
destroyed. It is marked released with a timestamp and left in the list. It is destroyed
only once the world clock has advanced past that timestamp by the grace period.

## State

```text
RECORD storage
  descriptions : list<description>
  # invariant: every entry is either live (referenced) or released-and-timestamped
  # invariant: a released entry younger than GRACE_PERIOD is still reusable — asking
  #            for its table identifier resurrects it rather than reparsing

CONSTANT GRACE_PERIOD = 300_000 ms   # five minutes of world clock
```

**Why five minutes.** The number is a bet about how a level plays: a creature leaves a
smart cover, fights or wanders, and comes back. Five minutes is long enough that a
round-trip across a level hits the cache, and short enough that a description for a cover
in a region the player has abandoned does not sit in memory for the rest of the session.
Nothing in the format forces it; a rebuild may tune it, but must keep *some* grace period,
because the failure it prevents — reparsing a table every time a creature re-enters cover —
is a visible frame-time spike.

## `description`

**Contract** — takes a configuration table identifier; returns a shared, reference-counted
description for it. Never fails: an unknown identifier causes a fresh parse, and a parse
failure is a hard error from the description itself, not a returned error here. Allocates
only on a miss. Runs a garbage sweep first, so a call is not constant-time. Not
thread-safe; it is called from the simulation thread only.

```text
FUNCTION description(table_id) -> description
  collect_garbage()                      # do this FIRST: see note
  FOR EACH d IN descriptions
    IF d.table_id IS table_id
      RETURN d                           # resurrects a released-but-young entry
  d = parse description from table_id
  APPEND d TO descriptions
  RETURN d
```

**Notes** — the sweep runs *before* the lookup, never after. The ordering is what makes
the grace period observable rather than merely theoretical: a description that expired is
gone by the time the lookup runs, so the lookup cannot hand out a corpse; and a
description that has not expired is still in the list, so the lookup finds it and revives
it. Reversing the two would let a caller receive an entry that the same call then deletes.

Identity is compared by the *interned* identity of the table name, not by string content.
Names here come from one interning table, so pointer equality is both correct and the
point — a rebuild with interned strings gets this for free; one without must compare text.

## `collect_garbage`

**Contract** — removes and frees every description that has been released and whose
release timestamp is older than the grace period. No return value, no failure mode. Walks
the whole list. Cheap in practice because the list holds tens of entries, not thousands.

```text
FUNCTION collect_garbage()
  REMOVE d FROM descriptions WHERE
      d.released AND (now - d.release_time) >= GRACE_PERIOD
  # removal destroys the description
```

**Invariants** — after the sweep, every remaining entry is either referenced or younger
than the grace period.

## `~storage` (teardown)

**Contract** — asserts in a checked build that every description was released before the
storage dies, then frees the whole list unconditionally. The assertion is the valuable
part: a description still referenced at shutdown means a cover, a creature's planner or a
script object outlived the storage, which is a lifetime-ordering bug in the level teardown
sequence rather than a leak to be tolerated.

**Notes** — the unconditional free at shutdown deliberately ignores the grace period.
Nothing is going to ask again.
