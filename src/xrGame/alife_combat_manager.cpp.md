# src/xrGame/alife_combat_manager.cpp

> Where off-screen combat was resolved, and now is not: all that survives is the routine that turns a creature killed off-screen into a lootable corpse in the right place.

**Needs** — [`alife_combat_manager.h`](alife_combat_manager.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`alife_simulator_base.h`](alife_simulator_base.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md)
**Used by** — [`alife_combat_manager.h`](alife_combat_manager.h.md)
**Tier floor** — T3: registry bookkeeping in a fixed order.

## Purpose

The off-screen simulation originally fought its own battles: two parties meeting on a
cross-level graph vertex detected each other, estimated their odds, traded abstract hits
until one side died or fled, and left the loser's inventory on the ground. In the shipped
engine that whole model is disabled — the source retains it commented out — and only one
piece of it is still reachable: killing one entity and leaving its body and belongings
where the simulation can find them later.

A rebuilder should read this file as *a slot*, not a feature. Off-screen creatures in the
shipped games die because a script or a smart terrain says so, not because the simulation
fought them.

## `kill_entity`

**Contract** — Converts a living off-screen creature into a corpse. Asserts the creature is
alive on entry — killing a corpse twice would double its loot. Leaves the creature's
belongings in a shared temporary list the caller is expected to drain.

```text
FUNCTION kill_entity(creature, graph_vertex_of_the_encounter, killer)
  REQUIRE creature is alive
  append the creature's children to the pending-items list
  old_vertex = creature's current graph vertex
  assign_death_position(creature, graph_vertex_of_the_encounter, killer)
      # decides where the body actually falls: near where it was met, offset
      # toward or away from the killer

  detach everything the creature carries
  REQUIRE the creature now has no children

  remove the creature from the scheduler            # a corpse gets no more updates
  IF the death position moved it to a different graph vertex
    remove it from the old vertex's registry
    add it to the new one
  IF the creature is itself carryable (a small creature that can be picked up)
    add it to the pending-items list too
```

**Invariants** — The order is the contract, and three steps of it are the engine-wide rule
from §6 that a destroyed or transformed entity must be unreferenced by every registry
before anything else happens to it:

1. The belongings are collected *before* they are detached, because detaching empties the
   list they are read from.
2. The assertion that the creature has no children after detaching is a real check, not
   decoration: a child left attached to a corpse is unreachable loot and a dangling
   reference in the object registry.
3. Scheduler removal precedes the graph move. A creature registered on the scheduler while
   its graph vertex changes underneath it can be updated at a vertex it no longer occupies.

The graph move is conditional on the death position having actually changed vertex. That is
not an optimization — removing and re-adding at the same vertex would trip the registry's
duplicate-key assertion.

**Notes** — A creature that is *also* an inventory item — the small animals the player can
pick up and carry — appends itself to the loot list after its own children. So a dead
carryable creature is simultaneously a corpse and an item, which is exactly what the game
wants.

## Construction

**Contract** — Takes the server and a configuration section and passes them to the
simulation base. The combat model's own setup — seeding the random generator from the
cycle counter, reading a maximum iteration count from that section, reserving the two
combat party buffers at 255 entries — is disabled along with the rest.

**Notes** — The manager still derives from a random-number source and still occupies a
layer of the simulation's inheritance chain, which is why it exists at all.

## The disabled combat model

Not reachable, but it is the reason for the file, the reason `kill_entity` has the shape it
has, and a description of a feature a rebuild may want. Its shape:

- **Detection.** An encounter is classified by what the two parties are — creature against
  creature, creature against anomaly, anomaly against creature, or creature against smart
  terrain — and each classification has its own detection probability drawn from the tuned
  evaluation functions. Detection is asymmetric and evaluated twice, once per direction;
  which side detected first decides who is the attacker, and mutual detection is recorded
  separately. Two *friendly* humans that meet are an interaction but not a fight; two
  friendly non-humans are not an interaction at all.
- **Odds.** Each side estimates a victory probability per pairing and multiplies along
  until the running product falls below the estimating creature's configured retreat
  threshold; whichever side runs out of members first loses. That is a cheap approximation
  of a battle's outcome that costs one evaluation per pairing rather than a simulation.
- **Attack.** An attacker fires a configured number of times per round, each shot a
  success-probability roll, each hit landing on a *uniformly random* member of the other
  side for a damage drawn uniformly between half and one and a half times its weapon's
  power, scaled by the victim's immunity to that hit type. Ammunition is consumed and the
  round ends early when it runs out.
- **Aftermath.** Weapon ammunition is written back, the dead are removed from their groups,
  their death positions assigned, their inventories pooled, and the *winning* side attaches
  what it wants from the pool. When both sides died or both fled, nothing is taken.

**Notes** — The two combat parties are indexed 0 and 1 throughout and "the other side" is
written as the index with its low bit flipped. That idiom is all over the disabled code and
is worth keeping in a rebuild: it makes every routine symmetric in the two parties without
a parameter saying which is which.

Time in the simulation is a single integer of milliseconds, and the disabled diagnostic
decomposes it into years, months, weeks, days and clock time by repeated division with
fixed divisors — 1000 milliseconds, 60 seconds, 60 minutes, 24 hours, **7 days to a week,
4 weeks to a month, 12 months to a year**. That calendar is the game's, not the Gregorian
one, and it is the only place the engine's own units of time are written down.
