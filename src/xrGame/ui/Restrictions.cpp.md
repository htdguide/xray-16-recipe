# src/xrGame/ui/Restrictions.cpp

> The multiplayer buy menu's purchase rules: rank gates what you may buy, and per-rank group
> limits gate how many of each kind you may carry.

**Needs** — [`Restrictions.h`](Restrictions.h.md)
**Used by** — [`Restrictions.h`](Restrictions.h.md)
**Tier floor** — T3: two configuration-driven tables and linear lookups over them

## Purpose

Multiplayer progression is expressed entirely in configuration: five ranks, each unlocking
more items and raising how many of certain kinds you may carry into a round. This file is the
only place those rules are read and the only place they are answered. It is separate from the
buy menu because the same rules are consulted from the menu, from the item lists and from the
server side.

## State

```text
RECORD Restrictions                      # one process-wide instance
  rank        : int                      # 0..4; the rank every query is answered against
  initialised : bool
  groups      : map<text, list<text>>    # group name -> the item sections in it
  limits      : list<(text, int)> [6]    # per rank, the per-group carry limit
  rank_names  : text [5]                 # localized, for messages
```

The limits table has **six** rows for five ranks: index 5 is the *base* row, a set of
defaults every rank inherits. The layered construction below is why.

Invariants:

- Every item section named in any restriction list must belong to exactly one group. The
  construction asserts the uniqueness; nothing re-checks it later, and an item in no group
  makes its count query undefined.
- Every group must have a limit in every rank's row. Rank 0 inherits the base row, rank *n*
  inherits rank *n−1*'s, so a group named only in the base row still resolves at rank 4.
- `initialised` guards construction, which is idempotent: calling it twice is a no-op, not a
  reload. Nothing reloads these tables during a session.

## `InitGroups`

**Contract** — Builds both tables. Runs at most once. Order matters and is the decision:

```text
FUNCTION init_groups()
  IF initialised THEN RETURN
  initialised = true

  # 1. Groups first: every line of the groups section is
  #    "group name = comma-separated item sections".
  FOR EACH (name, list) IN config["mp_item_groups"]
    add_group(name, list)

  # 2. The base row, which every rank starts from.
  add_restrictions(row: BASE, config["rank_base"].amount_restriction)

  # 3. Then each rank in ascending order, each layered on its predecessor.
  FOR rank IN 0 .. 4
    add_restrictions(row: rank, config["rank_" + rank].amount_restriction)
    rank_names[rank] = localize(config["rank_" + rank].rank_name)
```

**Invariants** — Groups must exist before restrictions are added, because a restriction names
a group. Ranks must be built in ascending order, because rank *n* copies rank *n−1*.

### `add_restrictions` — the layering

**Contract** — Fills one row of the limits table. A non-base row **first copies** its
predecessor's row wholesale — rank 0 copies the base row, rank *n* copies rank *n−1* — and
then applies its own list as *overrides*: a `group:count` entry already present updates the
count, one not present is appended.

```text
FUNCTION add_restrictions(row, list)
  IF row is not BASE
    limits[row] = copy of limits[row = 0 ? BASE : row - 1]

  FOR EACH entry IN split(list)
    (group, count) = parse "name:count"
    IF limits[row] has group THEN set its count to count
    ELSE append (group, count)
```

**Notes** — Only the base row may *introduce* a group; every other row may only override one.
The original asserts this and the shipped data obeys it. That is what makes the table
complete at every rank without every rank having to list every group, and it is the reason
`rank_base` exists as a section at all.

A malformed entry — one with no colon, or a count that does not parse — is fatal. The format
is `section_name:count` and it is frozen by the shipped configuration.

## `get_rank`

**Contract** — Free function, not a method: given an item section, return the lowest rank
allowed to buy it. It reads a *different* configuration key from the restriction tables — each
rank section's `available_items` list — caches all five lists on first call, then returns the
index of the first list that mentions the section.

```text
FUNCTION get_rank(section) -> int
  IF cache is empty
    FOR rank IN 0 .. 4
      cache[rank] = config["rank_" + rank].available_items

  FOR rank IN 0 .. 4
    IF cache[rank] contains section THEN RETURN rank

  warn "no rank found for section"
  RETURN 0                 # unknown items are buyable by everyone
```

**Notes** — The containment test is a **substring** test against the whole comma-separated
list, not a tokenized membership test. That is a real hazard in the shipped data: an item
whose section name is a prefix of another's matches the shorter one's rank. The original
carries the bug; a rebuild should tokenize, and the shipped names happen not to collide.

An item found in no rank resolves to rank 0 with a warning rather than failing. That choice
was made deliberately (the source says so) to keep modified data sets loading; the stricter
original behaviour was a hard failure.

## `IsAvailable`

**Contract** — `get_rank(section) <= current rank`. One line, and the only purchase gate the
buy menu consults.

## `GetItemGroup`

**Contract** — Which group an item belongs to, by **linear scan of every group's item list**.
Returns empty when the item is in no group. Quadratic in the worst case and called per item
per menu refresh; the tables are small enough that the original never cared.

## `GetItemCount` / `GetGroupCount`

**Contract** — How many of an item's group — or of a named group — the current rank may carry.
`GetItemCount` resolves the item to its group first and then defers. Both fail loudly in
development builds when the group is unknown or has no limit for the current rank, because
both conditions mean the configuration is internally inconsistent and the buy menu would
otherwise silently let the player carry an unlimited number.

## `Dump`

**Contract** — Prints both tables to the log at construction in non-shipping builds, and
asserts along the way that every item has a group. It is a data-validation pass wearing a
diagnostic's clothing: it is the only thing that checks the configuration is complete. A
rebuild should keep the check and may drop the printing.
