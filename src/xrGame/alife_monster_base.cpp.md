# src/xrGame/alife_monster_base.cpp

> A creature's authored loot drop, decided once when it spawns, and the promotion hooks that route around its inheritance.

**Needs** — [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_monster_brain.h`](../xrServerEntities/alife_monster_brain.h.md) · [`xrServer.h`](xrServer.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one configuration read and a probability draw.

## Purpose

Non-human creatures carry nothing by default, but some kinds drop a part when killed — a
mutant's organ, a trophy. That drop is decided at spawn time, not at death time, so that it
is recorded in the save and survives an unload. This file makes that decision.

## `on_spawn`

**Contract** — After the base spawn handling, reads two optional configuration keys from the
creature's section and, with the stated probability, spawns one item into the creature's
inventory. A creature whose section names no item does nothing.

```text
FUNCTION on_spawn()
  base spawn handling
  IF the section has no spawn-item key THEN RETURN
  section     = the named item section
  probability = the configured probability
  roll        = uniform(0, 1)
  IF roll >= probability AND probability is not exactly 1 THEN RETURN
  spawn the item at this creature's position and vertices, parented to this creature
```

**Invariants** — The probability of exactly one is special-cased so that a certainty is a
certainty regardless of what the draw returns; without it a roll landing exactly at the top
of the range would skip a guaranteed drop. That is a real edge case for a draw over a closed
interval, and the shipped data does use a probability of one.

The item is spawned *as a child* of the creature, which is what makes it appear in the
corpse's inventory rather than on the ground. A rebuild must set the parent explicitly — the
spawn call does not do it.

**Notes** — The probability key is read unconditionally once the item key exists, so a section
naming an item without a probability fails at load rather than defaulting. That is the
engine's usual stance on half-authored data.

## `add_online` / `add_offline`

**Contract** — Route the promotion and demotion through a free function shared by several
creature classes, then notify the brain.

**Notes** — The shared implementation is reached as a free function rather than through the
inheritance chain because this class's bases do not agree on which transition to run — the
same diamond that
[`alife_creature_abstract.cpp`](alife_creature_abstract.cpp.md) resolves by naming a base
explicitly. Here the resolution is to hoist the body out of the hierarchy entirely, which is
the cleaner of the two answers and the one a rebuild should generalize: the transition is a
function of the record, not a method on it.
