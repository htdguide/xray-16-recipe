# src/xrServerEntities/smart_cast_stats.cpp

> Counts, in a debug build, which casts the table failed to answer — the tool the cast table was built with.

**Needs** — [`smart_cast_impl1.h`](smart_cast_impl1.h.md) · [`smart_cast.h`](smart_cast.h.md)
**Used by** — [`smart_cast_impl1.h`](smart_cast_impl1.h.md)
**Tier floor** — T3: a counted set of string pairs, printed sorted.

## Purpose

The cast table in [`smart_cast.h`](smart_cast.h.md) is a hand-maintained list of the casts
worth optimizing. This file is how that list is decided: every cast that falls through to
the general type query increments a counter keyed by its (source, target) pair, and a console
command prints the counters sorted by frequency. The top of that list is what belongs in the
table next.

It exists only in a debug build. In every other build the four entry points are absent and
the console commands report that the facility is off.

## State

```text
RECORD CastCount
  from  : text     # the source type's name, as the runtime reports it
  to    : text     # the target type's name
  count : int
```

**Invariants** — entries are kept in a set ordered by (from, to), so an increment is a
lookup rather than a scan. The ordering compares the **names' addresses**, not their
contents — the names come from the runtime's type information and the same type always
yields the same address within a process. That makes the comparison a pointer comparison and
the report's grouping correct, but it also means the ordering is meaningless across runs and
that a name arriving from elsewhere would create a duplicate entry. Nothing does.

## the two tallies

**Contract** — one tally counts only the casts that took the general query; the other, off
by default, counts **every** cast regardless of path. The first answers "what should be in
the table"; the second answers "how many casts does a frame actually perform", which is a
different and more alarming question. The second is off because collecting it is expensive
enough to distort the profile.

## `show`

**Contract** — prints every counted pair, ascending by count, with each pair's share of the
total; prints a congratulatory line if nothing was counted, which is the state the table is
aiming at. The header gives the number of distinct pairs and the grand total.

**Notes** — sorting ascending puts the worst offenders at the *bottom* of the output, which
is right for a console that scrolls.

## `clear` / `release`

**Contract** — reset the counters, and destroy the tallies at shutdown. Clearing between
two measured activities is how a single scene's casts are isolated from startup's.
