# src/xrGame/purchase_list.cpp

> Restocks a trader: reads an authored shopping list, rolls each line, spawns what came up, and records how short the roll fell so prices can react.

**Needs** — [`purchase_list.h`](purchase_list.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`GameObject.h`](GameObject.h.md) · [`Level.h`](Level.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: config iteration, a random roll, and a spawn request per item

## Purpose

A trader's stock is not authored item by item; it is authored as a *shopping list* — a
configuration section whose every line names an item section, how many to try for, and the
probability each attempt succeeds. Restocking rolls that list. The randomness is the point:
two playthroughs must not offer identical stock, and an item the dice refused should feel
scarce.

Scarcity is the second half of this file's job. Each line records a **deficit factor**: how
far the actual outcome fell short of the expected one. The trade system is meant to
multiply prices by it, so a line that rolled badly is both rarer and dearer.

## State

```text
RECORD PurchaseList
  deficits : map<section_name, real>    # section -> price multiplier, default 1.0
```

**Invariant** — a section appears at most once in a shopping list. A duplicate would
overwrite the first line's deficit silently, so it is rejected rather than merged.

## `process(ini_file, section, owner)`

**Contract** — restocks one inventory owner from one shopping-list section of a supplied
configuration file (which need not be the global one — quest scripts pass their own).
Discards the owner's useless stock first, clears any previous deficits, then rolls every
line. Spawns items into the world; does not block.

```text
FUNCTION process(ini_file, list_section, owner)
  owner.sell_useless_items()          # make room before deciding what to buy
  deficits.clear()

  FOR EACH (key, value) IN ini_file.section(list_section)
    IF value is empty THEN CONTINUE               # a bare key is a comment in practice
    IF no global config section named `key` THEN CONTINUE   # names an item that no longer exists

    count       = integer field 0 of value
    probability = field 1 of value, or 1.0 when the line has only one field

    roll_line(owner, key, count, probability)
```

**Notes**

- The key is the item's configuration section; the value is a comma-separated tuple. The
  one-field form (count only) is accepted with probability 1 because much of the shipped
  data is written that way. The original once demanded two fields and was relaxed.
- Skipping a line whose section does not exist is what lets one shopping list be shared
  across the three games, which ship different item sets.

## `process(owner, name, count, probability)` — rolling one line

**Contract** — attempts `count` spawns of one item section, each accepted with
`probability`, at the owner's position and navigation vertex, parented to the owner.
Records the line's deficit. Fails on a zero count, a zero probability, or a repeated
section.

```text
FUNCTION roll_line(owner, name, count, probability)
  position = owner.position
  vertex   = owner.navigation_vertex
  parent   = owner.id

  # Seeded from the cycle counter, so restocking the same trader twice in one
  # session gives different stock. Nothing here is meant to be reproducible.
  random = new generator seeded from the CPU cycle counter (low 32 bits)

  spawned = 0
  FOR i IN 0 .. count-1
    IF random.uniform(0,1) > probability THEN CONTINUE
    spawned = spawned + 1
    level.spawn_item(name, position, vertex, parent, ready_to_use: false)

  deficits[name] = count * probability / max(spawned, 0.3)
```

**Invariants** — every spawn is parented to the owner, so the item enters that trader's
inventory rather than landing on the floor, and is spawned *not ready to use* (a weapon
arrives unchambered).

**Notes**

- The deficit is the ratio of *expected* outcome to *achieved* outcome, so a line that got
  exactly what it expected yields 1 and a line that got nothing yields a large multiplier.
- The floor of `0.3` on the achieved count is what keeps a zero roll from dividing by zero,
  and simultaneously caps the multiplier at a little over three times the expected count.
  That cap is the real reason the constant exists; it is the most expensive an unlucky item
  can become.
- Seeding from the cycle counter makes restocking non-reproducible even from a save. That
  is deliberate for stock variety, but it means a rebuild cannot use this path anywhere
  determinism is required (the multiplayer server must not).
- **The deficit is computed and never consumed.** The trade price calculation reads a
  constant `1.0` where it once multiplied by this factor, with the real call commented out.
  So the scarcity pricing is dead in the shipped engine: stock still varies, prices no
  longer do. A rebuild should decide deliberately whether to revive it — reviving it is a
  one-line change and a visible balance change.
