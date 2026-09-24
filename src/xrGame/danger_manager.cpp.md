# src/xrGame/danger_manager.cpp

> One creature's threat list: turns raw perceptions into typed danger records, ages and prunes them, and names the single most urgent one for the brain to react to.

**Needs** — [`danger_manager.h`](danger_manager.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`memory_space.h`](memory_space.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`Actor.h`](Actor.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — reached through its declarations in [`danger_manager.h`](danger_manager.h.md); callers name that, not this file.
**Tier floor** — T3: list maintenance and a scoring function

## Purpose

A creature perceives things — it sees bodies, hears shots, registers hits. Most of that is
noise. This file is the filter that decides which perceptions constitute *danger*, keeps at
most one record per distinct threat, lets records decay, and produces a single winner each
update so the brain has exactly one thing to be afraid of.

It is one of the four managers that make up a creature's memory (alongside
[visibility](visual_memory_manager.h.md), [sound](sound_memory_manager.h.md) and
[hit](hit_memory_manager.h.md) memory), and it is deliberately downstream of all three: it
consumes their event records rather than sensing anything itself.

## State

```text
RECORD DangerManager
  objects   : list<DangerObject>    # at most one per (entity, type, sense) triple
  ignored   : list<entity_id>       # threats this creature has decided to stop
                                    #   reacting to. Persists across a save.
  selected  : optional<DangerObject>   # a view into `objects`; recomputed every update
  time_line : int                   # records older than this are discarded
  owner     : CustomMonster         # the creature; required, never replaced
```

**Invariant** — `selected` points *into* the list. Every mutation of the list must clear or
recompute it first; the removal paths do so explicitly. In a rebuild this should be an index
or a copy, since the aliasing buys nothing and is the file's only sharp edge.

**Invariant** — only `ignored` is persisted. The danger list itself is regenerated from
perception after a load, because a danger the creature can no longer perceive is no longer a
danger. The ignore list must persist, because it is a *decision* rather than an observation:
forgetting it would make the creature re-react to a body it already walked past.

## `update`

**Contract** — the per-frame pass. Prunes, then ranks. Allocates nothing beyond the list's
own compaction, does not block, and is called from the creature's memory update. It is also
called at the end of every removal, so the selection is never stale.

```text
FUNCTION update()
  # 1. prune
  drop each record for which expired_or_useless(record) is true
  # 2. rank: lowest score wins
  selected = none; best = +infinity
  FOR EACH record IN objects
    score = do_evaluate(record)
    IF score < best THEN best = score; selected = record
```

**`expired_or_useless`** is the prune predicate, and it has a side effect that is the whole
reason it is not a pure test:

```text
FUNCTION expired_or_useless(record) -> bool
  IF record.time < time_line THEN
    IF record names an entity AND record.type == fresh_entity_corpse THEN
      ignore(that entity)          # never react to this body again
    RETURN true
  IF record names no entity THEN RETURN false      # nameless records never expire here
  IF NOT useful(record) THEN
    IF record.type == fresh_entity_corpse THEN ignore(that entity)
    RETURN true
  RETURN false
```

**Notes**

- Promoting an expiring corpse sighting into a permanent ignore is what stops a creature
  from being startled by the same body every time it comes back into view. Without it, a
  patrol route past a corpse would re-trigger the reaction indefinitely.
- A record with no entity attached — a sound whose owner was never identified — is exempt
  from the usefulness half of the test and is pruned only by the time line. There is nothing
  to ignore it *by*, so the clock is the only lever.

## `do_evaluate`

**Contract** — scores a record. **Lower is more urgent**, because the selection takes the
minimum. Total; every type has a case and an unlisted type is a hard failure rather than a
default, so adding a danger type without scoring it is caught immediately.

```text
FUNCTION do_evaluate(record) -> real
  base = one of:
      grenade              1000     # most urgent
      enemy_sound          1000
      entity_attacked      2000
      attacked             2000
      fresh_entity_corpse  2250
      attack_sound         2500
      bullet_ricochet      3000     # least urgent
      entity_death         3000
  RETURN base * 10 + (now - record.time)
```

**Invariants** — the multiplier and the age term are one exchange rate and must be read
together: **ten milliseconds of age is worth one unit of base weight**. So the 1000-unit gap
between a grenade and a ricochet is overtaken only after the grenade's record is ten seconds
older than the ricochet's. That relation, not the individual numbers, is the tuning.

**Notes**

- The ranking is counter-intuitive on first reading: the *cheapest* score wins, so the
  numbers are inverse priorities. A rebuild that flips the comparison must also flip the
  table.
- The age term uses the engine's global clock against a timestamp the perception layer
  stamped with level time. They coincide in the shipped configuration; see the same note in
  [`danger_location.h`](danger_location.h.md).
- Ricochets and deaths score equal and worst. A bullet striking near you ranks *below* a
  grenade about to go off, which is the intended ordering, but it also ranks below a
  distant enemy merely making a noise. That looks wrong and may well be; no comment
  explains it.

## `add` — from a visible object

**Contract** — the visibility memory's hook. Ignores disabled records. Produces exactly one
kind of danger: a **fresh corpse**, and only for a living-entity record that is dead *and*
has a recorded killer. Seeing something that simply died of no recorded cause is not danger;
seeing something that was *killed* is.

## `add` — from a sound

**Contract** — the sound memory's hook, and the widest classifier in the file. Ignores
disabled records. The sound's type is a bitmask and the tests run in a fixed order, first
match wins:

```text
FUNCTION add_from_sound(event)
  IF event.sound_type HAS bullet_hit       THEN emit bullet_ricochet, sensed by sound; RETURN
  IF event.sound_type HAS weapon_shooting  THEN emit attack_sound,    sensed by sound; RETURN
  IF event.sound_type HAS injuring THEN
      # a hurt player is only alarming if the player is an enemy
      IF the sound's owner is the player AND the player is not my enemy THEN RETURN
      emit entity_attacked, sensed by sound; RETURN
  IF event.sound_type HAS dying            THEN emit entity_death,    sensed by sound; RETURN
  IF the owner is a living entity AND is my enemy THEN emit enemy_sound, sensed by sound
```

**Invariants** — the order is load-bearing: a sound can carry several of these bits at once
(a dying scream from someone being shot), and the first test that matches decides the danger
type. Moving `dying` above `injuring` would change which reaction a creature has to a
killing blow.

**Notes** — the player special-case is the only place a danger classifier asks *who*. A
friendly player crying out is not danger; an enemy player crying out is. The same exemption
is not made for non-player characters, so a friendly stalker being hurt does alarm the
creature. That asymmetry is not explained and may be an oversight.

## `add` — from a hit

**Contract** — the hit memory's hook. Ignores disabled records, zero-amount hits, and — the
important one — **hits on itself**. A creature being shot is not handled as *danger*; it is
handled by the enemy system, which has better information. Danger is what happens to
*others*. Everything surviving the filter becomes an `attacked` record sensed by hit.

## `add` — a finished record

**Contract** — the single entry point every classifier funnels into. Three decisions, in
order:

```text
FUNCTION add(record)
  IF I already have a selected enemy AND record names an entity THEN
      ignore(that entity)         # I am busy; stop tracking this as danger
  IF NOT owner.accepts(record) THEN RETURN      # per-creature-class veto
  IF a record equal to this one exists THEN replace it in place; RETURN
  append record
```

**Invariants** — replace-in-place rather than append is the de-duplication, and it works
because [equality is identity without position or time](danger_object_inline.h.md): a
repeated perception of the same threat refreshes the record's timestamp and position, which
is exactly what keeps it from ageing out while it is still happening.

**Notes** — the first clause is the system's main damper. Once a creature has picked an
enemy, everything that entity does is permanently demoted out of the danger channel, because
the enemy channel already drives the behaviour and two systems reacting to one entity fight
each other. The cost is that the ignore is permanent for the rest of the creature's life: it
is never revisited when the enemy is lost.

## `useful` · `is_useful` · `evaluate`

**Contract** — three predicates that look alike and are not.

- `useful` is this file's own test: a record is useful unless its entity is on the ignore
  list (and it has no dependent object), and unless its timestamp has fallen behind the time
  line. The dependent-object exemption matters: a grenade danger survives its thrower being
  ignored, because the danger is the grenade, not the man.
- `is_useful` and `evaluate` both delegate to the owning creature, which is where a
  particular creature class installs its own policy. A blind creature can refuse visual
  dangers here; a scripted one can rescore them. This file supplies the default scoring
  (`do_evaluate`) and the creature decides whether to use it.

## `remove` · `remove_links` · `ignore` · `reset` · `reinit`

**Contract** —

- `remove` drops one record by identity, clearing the selection first if it named that
  record, and re-ranks.
- `remove_links` is the object-destroyed sweep and must run before any object dies. It
  clears the selection if it named that object, drops every record about it, clears the
  *dependent* pointer on every record that pointed at it (without dropping those records —
  see [`danger_object.h`](danger_object.h.md)), and removes it from the ignore list so its
  identifier can be reused by a later entity. That last step is essential: entity
  identifiers are recycled, and a stale ignore entry would silently suppress danger from
  whatever spawns into the freed slot.
- `ignore` appends an identifier if it is not already present. Idempotent, unbounded, never
  trimmed.
- `reset` empties the list and the selection but *keeps* the ignore list and the time line.
- `reinit` clears everything, including the ignore list, and is the full re-spawn path.

## `save` · `load`

**Contract** — serializes the ignore list and nothing else, into the entity's save packet.
See the state invariant above for why that is the right cut.

## `Load` · `reload`

**Contract** — both empty. The danger system reads no configuration; every number in it is
compiled in. A rebuild should note that the scoring table is therefore *not* moddable,
which is unusual for this engine and is worth changing deliberately rather than by accident.
