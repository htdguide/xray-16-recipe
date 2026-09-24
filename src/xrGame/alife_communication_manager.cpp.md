# src/xrGame/alife_communication_manager.cpp

> Where two off-screen characters meeting on the world graph would have traded with each other, item for item, by solving a subset-sum problem over both inventories. The whole model is disabled; only the constructor remains.

**Needs** — [`alife_communication_manager.h`](alife_communication_manager.h.md) · [`alife_simulator_base.h`](alife_simulator_base.h.md)
**Used by** — [`alife_communication_manager.h`](alife_communication_manager.h.md)
**Tier floor** — T2 if revived: the search is exponential in the worst case and is written against a fixed-depth explicit stack; as shipped, T4.

## Purpose

Two characters meeting outside the loaded level should, in principle, swap what each wants
from the other and settle the difference in cash. This file held that negotiation. It is
commented out in its entirety and nothing calls it; the shipped games trade only through
the player-facing trade screen, and off-screen characters keep what they have.

A rebuilder gets one thing from this file: the *design* of a trade the simulation can run
without a person watching, and the reason it is hard. Everything below describes disabled
code and should be read as a proposal, not as behaviour to reproduce.

## `CALifeCommunicationManager` construction

**Contract** — Takes the server and the simulation's configuration section and forwards
both to the simulation base. Nothing else. The type exists only as a layer in the
simulation's inheritance chain.

## The disabled trading model

### The problem

Each character wants a *set* of items from the pool of both inventories — their equipment
choices, made by the same eight-category priority walk the live stalker re-equipment uses
(food, knife, sidearm, rifle, grenade, medical, detector, armour; see
[`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) for the surviving version). Those wants
collide: both may choose the same rifle. And the resulting exchange has to balance — the
value each side gives up, plus cash, must match what it receives.

### The shape of the solution

```text
FUNCTION perform_trading(a, b)
  # 1. Budgets. Each side's purchasing power is its cash plus the total value of
  #    everything it carries — the same "sellable is spendable" rule the live
  #    re-equipment uses.
  a.budget = cash(a) + total value of a's items
  b.budget = cash(b) + total value of b's items
  IF either budget is zero THEN RETURN          # nothing to trade with

  pool = a's items followed by b's items
  detach everything from both sides             # the pool is now ownerless

  # 2. Eight rounds, one per equipment category, each side choosing from the pool.
  #    A category where both want the same item is re-run with that item blocked for
  #    one side and then for the other, so a collision costs three passes, not a
  #    deadlock. The state machine's four cases are: both choose, only a chooses,
  #    only b chooses, and the collision resolution.
  FOR category IN the eight, in priority order
    each side chooses from the pool, spending against its budget
    IF the two chose overlapping items
      re-run with the contested items blocked, alternating who is blocked
    ELSE
      commit both sides' choices, remove them from the pool

  # 3. Settle. The balance is the value each side took from the *other's* original
  #    holdings, netted. If cash alone covers it, pay and stop.
  balance = value a took from b, minus value b took from a
  IF balance is zero THEN done
  IF the debtor's cash covers the balance THEN pay it and done

  # 4. Otherwise items must move back to settle. Find a subset of one side's items
  #    whose value, plus available cash, cancels the balance.
```

### The subset-sum search

**Contract** — Enumerates the distinct totals reachable by any subset of a character's
items, then finds an actual subset hitting a chosen total. The enumeration is an explicit
depth-first walk over an array-backed stack of (remaining-count, highest-index-allowed,
running-total) frames, capped two ways: the stack is a fixed 128 frames deep, and the
enumeration stops after 30 distinct totals.

```text
FUNCTION generate_sums(items) -> sorted distinct subset totals
  totals = { 0 }
  FOR size IN 1 .. count(items)
    push the frame (size, last index, 0)
    WHILE the stack is non-empty
      pop (remaining, max_index, running)
      IF remaining == 0
        insert running into totals, keeping them sorted
        IF totals reached the threshold of 30 THEN RETURN
        CONTINUE
      FOR index FROM max_index DOWN TO 0
        push (remaining - 1, index - 1, running + cost of items[index])

FUNCTION find_subset(items, wanted_total) -> optional<indices>
  # The same walk, pruned: a branch whose running total already reaches or exceeds
  # the wanted total cannot be extended usefully, so it is abandoned.
  ... returns the first subset whose total is exactly the wanted one
```

**Invariants** — The two caps are what make an exponential search affordable, and both are
approximations with visible consequences. Thirty distinct totals means a character with a
large inventory has most of its possible offers never considered; a 128-frame stack caps
the size of any single subset. Neither is checked against the inventory size, so a
sufficiently rich character silently trades worse than a poor one. Any revival should
replace the enumeration with a bounded dynamic-programming table over value, which is
polynomial and has no cliff.

**Notes** — The search generates subsets in *descending* index order at each level, which
combined with the value-descending sort means it finds high-value offers first. That is
deliberate: the first hit is returned, so the ordering of the search *is* the negotiation's
preference.

### Capacity and ordering

**Contract** — A candidate exchange is checked for capacity by *performing* it — moving the
identifiers across both inventories — asking each side whether it can still carry what it
holds, and undoing the move when either says no. The search then continues to the next
candidate subset.

**Invariants** — The undo restores items to their original owners but appends them at the
end, so inventory order is not preserved across a rejected candidate. Combined with the
sort keyed on *previous owner* used elsewhere in the file, that is a real source of
order-dependence: the outcome of the negotiation depends on how many candidates were
rejected before it. A rebuild should evaluate capacity without mutating.

The comparison used to order items for the choice rounds sorts by cost descending, and
breaks ties by preferring items the *chooser already owned*, then by entity identifier. The
tie-break on prior ownership is what stops two equal items from ping-ponging between the
two characters across rounds; the identifier tie-break is what makes the result
deterministic, which a simulation that must produce the same world on every replay
requires.

### Ownership bookkeeping

**Notes** — There are two implementations of every ownership move, selected by a build
switch: a slow one that goes through the graph registry's attach for each item, and a fast
one that rewrites the children lists directly and re-derives the inventory afterwards. The
source carries a note from the author to himself saying the fast path "doesn't suit the OOP
paradigm" — which is the honest summary. The decision that matters is that an off-screen
inventory change is *two* facts, the owner's child list and the item's parent, and any
rebuild must keep them in step or the save will restore an item owned by nobody.

## `communicate_with_customer`

**Contract** — The one entry point the rest of the simulation would have called: a character
visits a trader, sells what the trader will buy, is paid, and gets his personal data device
back at the end regardless of what else changed hands.

**Notes** — The final step — detaching the character's original device from the trader and
re-attaching it — exists because the device is identity, not inventory: a character who
traded his device away would lose the record of who he is. The live re-equipment code keeps
the same rule as a filter rather than as a restore (see
[`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md)), which is the better shape.
