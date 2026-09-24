# src/xrGame/alife_smart_zone.cpp

> How a smart terrain behaves when the offline simulation walks something into it: it is always awake, it never fights, and meeting it means asking it for a job.

**Needs** — [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: four fixed answers to the offline-encounter interface

## Purpose

A **smart terrain** is an entity like any other as far as the alife simulation is
concerned: it sits on the game graph, it is scheduled, and the offline encounter system
will ask it the same questions it asks a creature. Those questions — what is your best
weapon, what do you do when you meet this thing, are you active, what is your best
detector — assume a combatant. This file is the smart terrain's four answers, and each
is a decision about what a place *is*.

The class is declared with the other alife server objects; only these four behaviours are
compiled in the game module.

## `action_type` — what meeting a smart terrain means

**Contract** — asked by the offline encounter system when a schedulable entity and this
smart terrain are considered to have met.

```text
FUNCTION action_type(other, group_index, mutual_detection) -> MeetAction
  IF other.game_vertex == this.game_vertex
    RETURN SMART_TERRAIN          # the encounter is "arriving at a place"
  RETURN IGNORE
```

**Invariants** — the test is **same game-graph vertex**, not proximity and not a radius.
A smart terrain's reach offline is exactly one coarse vertex: a creature is either at the
place or it is not. That is why smart terrains are authored onto graph vertices, and why
two smart terrains on the same vertex are indistinguishable to an arriving creature.

The distinct *smart terrain* action kind is what separates this from combat and trade:
the encounter resolver routes it to job assignment rather than to the offline combat
rules. The other two parameters — a group index and whether the detection was mutual —
are part of the encounter interface and carry no meaning for a place.

## `active`

**Contract** — always true. A smart terrain never sleeps: it must be able to receive an
arriving creature at any time, whether or not anything is currently registered with it.

## `best_weapon`

**Contract** — none, and it clears the cached best-weapon slot on the way out.

**Notes** — clearing the cache is the only reason this is not simply "return nothing". The
cached slot is part of the shared creature record the smart zone inherits; leaving a stale
weapon reference there would let the offline combat code believe a *place* is armed. A
rebuild whose smart terrain does not inherit a combatant's fields deletes this entirely,
which is the better shape.

## `best_detector`

**Contract** — declared unreachable. Calling it is a programming error, diagnosed rather
than answered.

**Notes** — this is the clearest statement of the file's theme. Three of the four
questions have a defensible answer for a place; this one does not, and rather than
returning nothing and letting a caller act on it, the smart terrain asserts that it should
never have been asked. A rebuild with a narrower offline-entity interface never generates
the call, which is what the assertion is asking for.
