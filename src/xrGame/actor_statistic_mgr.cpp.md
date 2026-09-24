# src/xrGame/actor_statistic_mgr.cpp

> Keeps the player's scorecard: find-or-create a section, find-or-create a tally within it, add to it, and total it — with "not scorable" as a first-class answer.

**Needs** — [`actor_statistic_mgr.h`](actor_statistic_mgr.h.md) · [`actor_statistic_defs.h`](actor_statistic_defs.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`alife_simulator_header.h`](alife_simulator_header.h.md)
**Used by** — [`actor_statistic_defs.h`](actor_statistic_defs.h.md)
**Tier floor** — T3: linear lookup over a few dozen records on gameplay events

## Purpose

The scorecard's only interesting decision is how it *totals*. Everything else is
find-or-create over two nested lists. The totalling rule has to accommodate the fact that
some entries on the statistics screen are not numbers at all — "favourite weapon", "time
survived" — and those must not silently contribute zero to a sum that is displayed next to
them.

## State

Owns nothing directly. The sections live in an alife registry — persistent, keyed to the
player, saved with the game — reached through a wrapper the manager creates at construction
and destroys with itself. The registry is initialized against key zero, since there is one
player.

## `GetSection` / `GetData`

**Contract** — find the section (or, within a section, the tally) with the given key, and
create an empty one at the end of the list if there is none. Never fails; never returns
nothing. A newly created tally starts at zero count and zero points with an empty text
value.

**Invariants** — creation appends, so list order is first-touch order, and that order is
the save order and the display order. A rebuild that sorts the list changes what the
statistics screen looks like on an existing save.

## `AddPoints` (numeric)

**Contract** — adds a count and a per-unit score to a tally, creating the section and the
tally if needed. The count accumulates the number of events; the points accumulate the
count times the per-unit score, so a caller reporting "three kills worth five each" leaves
a tally of count three and points fifteen.

```text
FUNCTION add_points(section_key, tally_key, count, points_each)
  tally = section(section_key).tally(tally_key)
  tally.count  = tally.count + count
  tally.points = tally.points + count * points_each
```

## `AddPoints` (textual)

**Contract** — sets a tally's displayed text, creating the section and the tally if needed.
Replaces rather than appends. Setting it makes the whole enclosing section unscorable.

## `GetTotalPoints` (a section's own total)

**Contract** — sums the section's tallies, or reports *not scorable* if any tally in it
carries a text value.

**Invariants** — "not scorable" is signalled by a negative one. The sentinel is not a
score: every real score is non-negative, and the caller must test for it rather than
adding it. This is the file's central convention and it propagates into the aggregate
total below.

```text
FUNCTION section_total(section) -> int
  total = 0
  FOR EACH tally IN section.data
    IF tally.text_value is non-empty THEN RETURN NOT_SCORABLE
    total = total + tally.count * tally.points
  RETURN total
```

**Notes** — the per-tally term multiplies the count by the accumulated points, and the
accumulated points already include the count (see the numeric adder above). So a tally
built by a single call of three kills at five each totals forty-five, not fifteen. Either
the adder should not pre-multiply or the totaller should not multiply again; the source
gives no way to tell which was intended, and the shipped score values were presumably
tuned against whatever it does. **A rebuild should decide deliberately and say so**, and
should expect existing saves' displayed totals to change.

## `GetSectionPoints`

**Contract** — the score for one named section, except that the reserved name `total`
means *the sum of every scorable section*. Sections that are not scorable are skipped
rather than poisoning the sum, and if no section is scorable the answer is itself the
not-scorable sentinel.

```text
FUNCTION section_points(key) -> int
  IF key != "total" THEN RETURN section(key).total

  aggregate = NOT_SCORABLE
  FOR EACH section IN storage
    value = section.total
    IF value != NOT_SCORABLE THEN
      IF aggregate == NOT_SCORABLE THEN aggregate = 0
      aggregate = aggregate + value
  RETURN aggregate
```

**Invariants** — the aggregate starts at the sentinel and is promoted to zero on the first
scorable section. That is what distinguishes "every section is textual" from "every
section scored zero", and the statistics screen renders the two differently.

**Notes** — `total` is a reserved section name. A caller that creates a section actually
named `total` finds it shadowed by the aggregation, and the find-or-create path will have
created an empty section that is never read. The reservation is undocumented in the
source.
