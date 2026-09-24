# src/xrGame/ui/UIInventoryUtilities.cpp

> The chapter's shared services: the icon atlases everything draws from, the game clock rendered as text, the belt-fit packing test, the threshold tables that turn a rank or a reputation number into a word, and the two hooks by which a UI act becomes a game event.

**Needs** — [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`Inventory.h`](../Inventory.h.md) · [`InventoryOwner.h`](../InventoryOwner.h.md) · [`Actor.h`](../Actor.h.md) · [`Level.h`](../Level.h.md) · [`date_time.h`](../date_time.h.md) · [`InfoPortion.h`](../InfoPortion.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md)
**Tier floor** — T3.

## Purpose

A grab bag by name and not by accident: these are the things **more than one screen needs and
none of them owns**. A rebuild is free to split it, and should keep the groupings — atlases,
clock formatting, belt packing, threshold naming, game notification — because each is a
distinct decision with its own rationale.

## State

```text
# five lazily created icon atlas materials, process-wide:
#   buy menu, equipment icons, multiplayer character icons,
#   outfit upgrade icons, weapon upgrade icons
# three lazily loaded threshold tables:
#   rank names, reputation names, goodwill names
#   each a map from an upper bound to a localization identifier
```

**Invariants**

- Atlas materials are created **on first use, not at startup**, and released together. The
  multiplayer and upgrade atlases are never loaded in a session that does not open those
  screens, which matters because they are large.
- The threshold tables are loaded once and shared; they are cleared explicitly at shutdown,
  not on a level change, because they come from static configuration.

## Icon atlas addressing

**Contract** — five named atlases, each one texture drawn through the same material.
Everything in this chapter that shows an item icon resolves a sub-rectangle of one of them.

**Notes** — the addressing convention is the frozen part, and it is stated here because the
header is where the grid constants live: an item's configuration section names its icon in
**grid cells**, and a cell is 50 by 50 canvas units for equipment and 64 by 64 for character
portraits. Character portraits are 2×2 cells for the inventory and trade screens and 2×5 for
the full-length view. Those numbers are in the shipped data's coordinates, so they are frozen
with it.

Trade screen icons are drawn at four fifths scale. One constant, one screen, no recoverable
reason beyond the layout.

## Belt packing

**Contract** — *would this item fit on the belt alongside what is already there?*

```text
FUNCTION fits_on_belt(existing, candidate, width, height) -> bool
  grid := width × height of free flags
  trial := existing + candidate
  sort trial by descending footprint          # widest first, then tallest
  FOR EACH item IN trial
    scan the grid row by row for the first position whose rectangle is free
    IF none THEN RETURN false
    mark that rectangle occupied
  RETURN true
```

**Invariants** — the sort is **strictly total**: items of equal footprint are ordered by
configuration section name and, within a section, by entity identifier. Without that the
answer would depend on the incoming order and the same belt could accept an item one moment
and refuse it the next.

**Notes** — **widest-first is the decision.** This is first-fit-decreasing bin packing, and
it is deliberately *not* the same algorithm the cell board uses to actually place items — the
board places in arrival order and grows or compacts. The two can disagree: this predicate can
say yes about an arrangement the board will not reach on its own, which is why the board also
has a compaction step. A rebuild must keep the sort, because the predicate's job is to
answer "is there *an* arrangement", not "will the board find it".

The candidate is appended to the caller's live list, tested, and then removed again. A
rebuild copies instead.

## The clock as text

**Contract** — split a game timestamp into calendar fields and format it at a requested
precision. Time precisions: hours; hours and minutes; with seconds; with milliseconds; and a
day-count form (`Nd hh:mm:ss`) for elapsed durations. Date precisions: year; month and year;
day, month and year. The month may be a localized name or a number, and both separators are
caller-supplied.

**Invariants** — a "not full" rendering **drops leading zero units**: at under an hour it
shows minutes and seconds, at under a minute just seconds. That is for durations, where
`00:00:07` reads wrong.

**Notes** — month names are a fixed twelve-entry table of localization identifiers, indexed
from the split month minus one. The separator being a parameter — a comma for dates, a colon
for times, by default — is why the same two functions serve the PDA's clock, the task list's
timestamps and the statistics screen's durations.

## Elapsed periods

**Contract** — render the gap between two timestamps as **one** unit: months if the months
differ, else days, else hours, else minutes, else seconds.

**Notes** — the coarsest differing unit wins and the rest are discarded, so "1 month" may
mean anything from a day to two months. That is the intended reading for a task's age.

The month arithmetic is wrong: it adds the year difference in months to the end month and
then subtracts the start month, which is not the month gap, and it computes the year
difference in a byte. For the in-game timescale, where a playthrough spans weeks, the result
is usually right by coincidence. Recorded as a defect a rebuild should simply not reproduce.

## Weight display

**Contract** — two renderings of carried versus maximum weight. One produces a single string
with **inline colour markup** — red when over the limit, otherwise the accent colour — and an
optional localized prefix. The other splits it across two widgets, carried and maximum, with
a localized unit.

**Notes** — the first embeds chapter 15's inline colour markup directly in the composed
string, which is what makes an overloaded player's weight turn red without the caller knowing
anything about colour. The two renderings exist because two screens' layouts differ; neither
is more correct.

## Threshold tables

**Contract** — a rank, a reputation or a goodwill number is turned into a word by a table
loaded from configuration as alternating (name, upper bound) pairs. The word is the first
entry whose bound **exceeds** the value; a value past every bound takes the last word.

**Invariants** — the configuration list must have an **odd** element count. The final entry
is a name with no bound, and its bound is synthesised as one past the previous — the
catch-all. A list with an even count is rejected.

**Notes** — encoding the catch-all as "the last name has no number" is why the parity check
exists, and it is the kind of thing that looks like an off-by-one until you see the intent.
A rebuild with an optional bound expresses it directly.

Lookup is strictly-greater, so a value exactly on a boundary takes the *higher* band. The
shipped tables are authored with that in mind.

## Colours from relations

**Contract** — goodwill, reputation and relation each map to one of three colours: green
above a threshold, red below the negative of it, grey between. The thresholds are ±1000 for
goodwill and ±50 for reputation; relation is already a three-valued enumeration.

**Notes** — three-valued and hard-coded, where the ranks and reputations *names* are
configurable. The source notes that these should interpolate across the range instead.
Recorded as an acknowledged simplification.

## Telling the game about a UI act

**Contract** — two one-way notifications, both no-ops outside single player:

- **an information portion** is transferred to the actor by identifier, which is the game's
  own fact-recording mechanism and therefore reaches tutorials, task preconditions and
  dialogue conditions alike;
- **a script call** announcing that the conversation screen was shown or hidden, encoded as
  a small integer mode.

**Notes** — this is the chapter's event-not-call rule at its most literal, and the first form
is the more interesting: a button press becomes an *information portion*, the same object a
dialogue line or a script would produce. The tutorial system listens for exactly those, which
is how a tutorial step can wait for "the player opened the map tab" without anything in the
UI knowing about tutorials.

The second form is a bare pair of magic numbers, 10 for shown and 11 for hidden, matched
against two hard-coded string identifiers. There is no enumeration and no registry; both ends
agree by convention. Recorded as unrecovered rationale for the numbers.

## `CreateShaders`

**Contract** — does nothing. Every atlas is created on demand. Kept as an entry point because
the startup sequence calls it.
