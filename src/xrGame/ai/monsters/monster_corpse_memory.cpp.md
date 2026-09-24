# src/xrGame/ai/monsters/monster_corpse_memory.cpp

> The dead bodies a creature has seen recently and still considers worth eating, each with where and when it was seen, kept fresh by an eviction pass and queried as "the nearest one".

**Needs** — [`monster_corpse_memory.h`](monster_corpse_memory.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md) · [`memory_manager.h`](../../memory_manager.h.md) · [`visual_memory_manager.h`](../../visual_memory_manager.h.md) · [`item_manager.h`](../../item_manager.h.md)
**Used by** — [`monster_corpse_memory.h`](monster_corpse_memory.h.md)
**Tier floor** — T3: a keyed collection over entity handles with a linear scan for the nearest

## Purpose

Scavenging needs a creature to remember bodies it is not currently looking at, and to stop
remembering ones that have been picked clean, claimed by another creature, or seen too long
ago. This holds that set.

The memory is fed from the creature's general *item* perception rather than from a corpse
notification: every remembered object that is visible right now is examined, and the ones that
are entities and not alive are admitted. So a creature only learns about a corpse by seeing it,
never by hearing about it or by walking over it blind.

Two facts are recorded per corpse beyond its identity: **where** it was (a position and a
navigation vertex, so the creature can path to it without re-acquiring it) and **when** it was
last seen. The position is a snapshot, deliberately — a creature that saw a body dragged away
still walks to where it last saw it.

## State

```text
RECORD CorpseSighting
  position   : vector       # where the body was when last seen — a snapshot, not a live read
  vertex     : int          # the navigation vertex the body occupied
  time       : int          # global clock at that sighting

RECORD CorpseMemory
  creature   : BaseMonster
  retention  : int          # milliseconds a sighting survives; default 10000, overridden at bind
  sightings  : map<entity, CorpseSighting>
```

Invariants: every key in `sightings` is a dead entity, is not destroyed, still has food left,
and is not claimed by another creature — the eviction pass restores all four every tick, so a
reader between ticks may see an entry that has just become invalid. `best_corpse` re-checks the
claim for exactly that reason.

## `update`

**Contract** — the per-tick step. Admits every currently visible dead entity from the
creature's item perception, then evicts everything that has gone stale. Called once per
creature update; allocates only when a new corpse is admitted.

```text
FUNCTION update()
  FOR EACH object IN creature.remembered_items()
    IF NOT creature.sees_now(object) THEN CONTINUE
    IF object is not an entity OR object is alive THEN CONTINUE
    remember(object)

  evict_stale()
```

**Notes** — the visibility test is "seen now", the *recent* sense of vision rather than the
strict this-instant one, so a corpse briefly occluded is still re-stamped with a fresh time.
That is what stops a creature forgetting a body it is standing over while its head turns.

## `remember`

**Contract** — insert or refresh one corpse's sighting with the current position, vertex and
clock. Silently does nothing for a corpse another creature has claimed. Refreshing overwrites
unconditionally: the newest sighting always wins.

```text
FUNCTION remember(corpse)
  IF corpse.is_claimed() THEN RETURN          # another creature is already eating it
  sightings[corpse] = { corpse.position, corpse.navigation_vertex, now }
```

## `evict_stale`

**Contract** — remove every entry that fails any of five conditions. Called from `update`.

```text
FUNCTION evict_stale()
  FOR EACH (corpse, sighting) IN sightings
    IF corpse is missing
       OR corpse has come back to life
       OR corpse is being destroyed
       OR sighting.time + retention < now
       OR corpse.food_remaining < 1          # picked clean
      remove the entry
      CONTINUE
    IF corpse.is_claimed()                   # another creature got there first
      remove the entry
```

**Notes** — the food threshold is the same one the corpse manager uses to abandon a forced
target, so "there is nothing left to eat" is a single shared rule. Whether a body is *claimed*
is a live query on the body, not a remembered fact, which is how two creatures avoid converging
on one meal without any coordination between their memories.

The two removal branches are separate only because the claim test needs to mutate the corpse's
representation to ask; merge them in a rebuild.

## `best_corpse` / `best_corpse_info`

**Contract** — the nearest remembered corpse and its recorded sighting. `best_corpse` answers
nothing if the nearest one has been claimed since the last eviction pass — note that it does
*not* then fall back to the second-nearest, so a claimed near corpse hides a free far one for
one tick. `best_corpse_info` answers a sighting with time zero when the memory is empty, which
is how callers detect "no corpse" from the info alone.

```text
FUNCTION find_nearest() -> optional<entry>
  best = none; best_distance = infinity
  FOR EACH (corpse, sighting) IN sightings
    d = straight_line(sighting.position, creature.position)
    IF d < best_distance THEN best_distance = d; best = the entry
  RETURN best
```

**Notes** — nearest is measured from the *remembered* position, not the corpse's live one, so a
body that has been moved is ranked where the creature thinks it is. That is consistent with the
creature then walking there.

Ranking is purely by distance: there is no preference for a fresher sighting, a fuller body or
one the creature is facing. A rebuild is free to add one, but the shipped behaviour — creatures
converging on the closest thing regardless — is what the levels were populated against.

## `forget_entity`

**Contract** — drop one entity from the memory, used when that entity leaves the world. Stops
at the first match; the collection is keyed by identity so there is at most one.
