# src/xrGame/alife_trader.cpp

> The concrete trader server object: an entity that is both a character and a container, and the one place a price is asked for.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`alife_trader_abstract.cpp`](alife_trader_abstract.cpp.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`ai_space.h`](ai_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two delegations and one lookup

## Purpose

A trader's server object inherits from two parents — the dynamic-object side that gives it
a place in the world, and the trader side that gives it an inventory — and this file
exists to resolve that diamond for three operations where both parents have an opinion.
Everything of substance is in
[`alife_trader_abstract.cpp`](alife_trader_abstract.cpp.md); a rebuild that composes
rather than inherits deletes this file and keeps only the price function.

## State

`Stateless.`

## `spawn_supplies`

**Contract** — run *both* parents' supply spawn, in declaration order: the generic
dynamic-object supply list from the entity's configuration section first, then the
character-profile supply list.

**Notes** — the order is the only decision in the function, and it is observable: the
profile's items are appended after the section's, and inventory order determines which of
two equivalent items a trader offers first.

## `dwfGetItemCost`

**Contract** — the price of one inventory item held by this trader. Returns the item's own
recorded cost.

**Invariants** — none. The returned value does not depend on anything the function
computes.

**Notes** — **this function does not do what it looks like.** It identifies whether the
item is an artefact, and if so counts how many artefacts with the same section name this
trader already holds — and then returns the item's unmodified cost regardless, discarding
the count. Two notes left in the source name both halves as unfinished: correct pricing
for non-artefact items, and a data structure that would make the count cheap.

So the intent is recoverable — artefact prices were meant to fall as a trader accumulated
duplicates, the way the series' economy is usually described — but **the rule itself is
not**: no formula was ever written. A rebuild should implement a plain cost lookup, which
is what the shipped game actually does, and treat the scan as dead code rather than
reproducing a loop whose result is thrown away.

## `add_online`, `add_offline`

**Contract** — promotion and demotion of this trader's inventory. Both forward to the
trader side of the inheritance and discard the dynamic-object side's version.

**Notes** — the choice of parent is the whole content of these two functions. The
dynamic-object path would move the trader's children as generic children; the trader path
knows they are inventory items and performs the identifier reassignment and the
must-not-be-saved cull described in
[`alife_trader_abstract.cpp`](alife_trader_abstract.cpp.md). Picking the wrong one loses
inventory across the boundary.
