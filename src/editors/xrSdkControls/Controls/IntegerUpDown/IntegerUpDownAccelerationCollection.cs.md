# src/editors/xrSdkControls/Controls/IntegerUpDown/IntegerUpDownAccelerationCollection.cs

> The acceleration schedule, kept sorted by hold time so the spin box can walk it forward and never backward.

**Needs** — [`IntegerUpDownAcceleration.cs`](IntegerUpDownAcceleration.cs.md)
**Used by** — [`IntegerUpDown.cs`](IntegerUpDown.cs.md)
**Tier floor** — T3: an ordered list.

## Purpose

A list of [acceleration entries](IntegerUpDownAcceleration.cs.md) that maintains one invariant: **ascending order by hold time**. That invariant is the entire reason the type exists rather than a plain list, because [`IntegerUpDown`](IntegerUpDown.cs.md) advances through it by index and compares only against the *next* entry. An unsorted table would make promotions arbitrary.

## State

```text
RECORD AccelerationCollection
  items : list<Acceleration>    # invariant: non-decreasing in `seconds`
```

## `add`

**Contract** — inserts at the first position whose hold time exceeds the new entry's, so equal hold times keep insertion order. Rejects an absent entry.

```text
FUNCTION add(entry)
  position = first index i WHERE entry.seconds < items[i].seconds, else end
  insert entry at position
```

**Notes** — a linear scan, chosen over a binary search because the table is a handful of entries and is built once. Saying so is the point: it is a deliberate non-optimization, not an oversight.

## `add_all`

**Contract** — adds several entries at once, each through the sorted insert. **Validates the whole batch before inserting any of it** — one absent entry rejects the call and leaves the collection untouched — so a partially applied schedule is impossible.

## The rest of the surface

**Contract** — read by position; count; contains; remove by value; remove all; copy out; iterate. Read-only by position: an entry is replaced by removing and adding, because assigning by position would break the ordering invariant.
