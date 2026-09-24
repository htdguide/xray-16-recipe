# src/xrGame/alife_human_object_handler_save.h

> The real offline inventory manager, preserved and not compiled: how an off-screen character noticed items lying on the world graph, decided what to keep, merged its ammunition and weighed what it could carry.

**Needs** — _(none; nothing here compiles)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4 as shipped; T3 if revived.

## Purpose

Every method stubbed in
[`alife_human_object_handler.cpp`](alife_human_object_handler.cpp.md) has its real body
here, commented out and marked do-not-delete. The behaviour it describes — off-screen
characters wandering the world graph, spotting loot, equipping themselves and merging their
ammunition — is the part of the *alife* promise the shipped games do not actually keep.

This page records the design, for a rebuild that wants it and for a reader wondering why the
stubs exist. None of it runs.

## The item-capacity model

```text
# Carried on every human record:
  cumulative_item_mass   : real     # sum of what it carries
  max_item_mass          : real     # its configured limit
  cumulative_item_volume : int      # sum of inventory grid areas
  # a global maximum volume applies to everyone

FUNCTION can_get_item(item) -> bool
  RETURN cumulative_mass + item.mass <= max_item_mass
     AND cumulative_volume + item.volume <= the global volume limit
```

**Invariants** — Offline inventory capacity is *two* independent limits, mass and volume,
and volume is computed from the item's inventory-grid footprint — the same width and height
the player's inventory screen lays out. The offline and online models therefore agree on
what fits, which is what makes a character's inventory survive the boundary unchanged.

## Noticing items — `process_items`

**Contract** — Scans every object indexed at the character's current graph vertex, keeps the
ones that are useful inventory items and currently offline, and admits each with an
independent draw against the character's detection probability. Anything admitted goes into
a shared candidate list, and the equipment pass runs if the list is non-empty.

**Invariants** — Items already online are skipped: a live object on the loaded level belongs
to the player's world, not to the offline simulation's. Detection is per item, not per
scan, so a character walking over a pile takes some of it and leaves the rest.

## Weighing candidates — the shared list and the sort

**Contract** — Candidates are sorted by cost, most valuable first, before any choice is made.
A second ordering by inventory-grid area exists for the volume-driven case.

**Invariants** — Sorting by value descending is what makes the greedy equipment walk
approximately right: every category's chooser takes the first acceptable candidate it
sees, so the ordering *is* the preference within a category.

## Choosing — `attach_items`

**Contract** — The whole equipment decision, in three shapes selected by a *take type*.

```text
FUNCTION attach_items(take_type)
  REQUIRE the character is alive           # a corpse cannot pick things up
  IF this is a group record
    divide the loot among the members in two passes (minimum first, rest second)
    RETURN
  IF everything fits without any choosing at all
    take it all and RETURN                 # the fast path

  IF take_type is "all"
    pool the character's own inventory into the candidate list and drop everything,
    so the choice is made over old and new together
  sort candidates by value, descending

  IF take_type is "all" or "minimum"
    choose_food()
    choose_weapon(Knife); choose_weapon(Secondary)
    choose_weapon(Primary); choose_weapon(Grenade)
    choose_medikit(); choose_detector(); choose_equipment()
  IF take_type is "all" or "rest"
    choose_valuables()                     # take whatever still fits, by value
```

**Invariants** — The priority order is the same eight-category order the live re-equipment
uses (see [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md)), and for the same reason:
budget and capacity are consumed in priority order.

The *fast path* is the interesting optimization: before choosing at all, the routine
simulates taking everything and checks whether mass and volume still fit. When they do, no
choice is needed and everything is taken. That is the common case for a character walking
past two items, and it skips the entire evaluation machinery.

The three take types exist because loot is divided in two rounds: **minimum** equips
everyone, **rest** distributes the remainder, **all** is the single-character case that does
both and also reconsiders what the character already carries.

## The choosers

**Contract** — Each of the eight is the same shape: walk the candidates, skip those the
character cannot afford or cannot carry, skip those already claimed by the other party when
one is given, score the rest with the matching tuned evaluation function, and take the best
— up to a per-category count limit for the stackable categories.

**Invariants** — Money is *restored* after each chooser runs. The budget is spent during the
walk so that later candidates in the same category see a reduced purse, and then rolled back
so the next category starts from the full amount. That is deliberate and it means the
categories do not compete for money — each is limited by the whole purse. A rebuild wanting
categories to compete simply stops restoring.

The armour chooser is disabled with the same "due to the game design" comment as the live
one: authored equipment stays authored.

Each chooser has a *two-mode* behaviour on success, selected by whether a claimed-item list
was passed: with no list it attaches the item for real, through the world index; with a list
it only records the choice as a pending child. That is how a two-party division proposes
before it commits.

## Ammunition

Three routines and one merge.

**Contract** — `get_available_ammo_count` sums the rounds in every inventory item whose
section name is a substring of the weapon's accepted-ammunition list. A weapon with no
ammunition list reports the maximum count, meaning *unlimited*.

**Invariants** — The match is a substring test on section names, not a set membership test.
That is a real, load-bearing shortcut: the accepted list is one text field of comma-joined
section names, and the check is whether the candidate's name appears anywhere in it. It
gives false positives for any section whose name is a prefix or infix of another, and the
shipped data avoids them by naming convention. A rebuild should parse the list once and
compare properly.

**Contract** — `attach_available_ammo` takes matching ammunition for a weapon, affordable and
carryable, up to a small fixed count, then restores the purse.

**Contract** — `update_weapon_ammo` writes a weapon's spent rounds back into the inventory
after combat: it walks the matching ammunition items, draining the weapon's remaining count
against each in turn, and destroys those that end up empty.

```text
FUNCTION collect_ammo_boxes()      # merge partial boxes of the same ammunition
  FOR EACH pair of unmerged matching ammunition items
    IF their combined rounds exceed one box
      fill the first to exactly one box; the second keeps the remainder,
      and becomes the accumulator for the rest of the pass
    ELSE
      pour the second into the first; the second is left empty
  destroy every item left with no rounds
```

**Invariants** — The merge pass fills boxes to exactly their configured capacity, never past
it, and it carries the *partial* box forward as the accumulator so that at most one partial
box of each kind survives. That is the whole point: an offline character that fought several
times would otherwise accumulate dozens of nearly-empty boxes, each a separate entity
against the world's 16-bit identifier budget.

The pass marks items as visited in a shared scratch array owned by the simulation rather
than a local one — a global reused buffer, sized to the largest inventory. A rebuild uses a
local set.

## Choosing a weapon and a detector

**Contract** — `best_weapon` walks the inventory, refreshes each weapon's available round
count, and keeps the highest *weapon class* among those that have ammunition or are in the
natural-weapon or throwable slot. If nothing qualifies it falls through to the trader base's
answer.

**Invariants** — The weapon class number is used directly as the ranking, so the numbering in
the configuration is a preference order, not just a taxonomy. Two things bypass the
ammunition requirement: the natural-weapon slot and the throwable slot, because neither
draws from the inventory.

**Contract** — `best_detector` prefers a *visual* detector over a *simple* one and returns
immediately on finding one; for a group record it instead scores each member's own best
detector and keeps the highest.

**Notes** — The group branch reads member index zero on every iteration rather than the
current member — a plain bug, which would make a group's best detector always the first
member's. Anyone reviving this must fix it.

## `detach_all`

**Contract** — Drops everything, in one of two modes. The *real* mode routes each item
through the world index so it ends up lying on the character's graph vertex. The
*fictitious* mode only unlinks it, leaving the item indexed nowhere — used when the items
are about to be re-attached to somebody else and must not briefly exist on the ground.
Asserts that the character's cumulative mass and volume end at zero, which is the check that
the two accumulators stayed in step with the inventory all along.
