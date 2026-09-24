# src/xrGame/hit_memory_manager.cpp

> The sense of being hurt: a bounded list of who has damaged this creature, how hard, from what direction, and when — the input the brain's danger model reads.

**Needs** — [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`memory_space_impl.h`](memory_space_impl.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`client_spawn_manager.h`](client_spawn_manager.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — reached through its declarations in [`hit_memory_manager.h`](hit_memory_manager.h.md); callers name that, not this file.
**Tier floor** — T2: a bounded list with an eviction policy and a deferred-resolution load path

## Purpose

Of the three senses, this one is the simplest to describe and the hardest to get right,
because a hit record refers to *another entity* and that entity may not exist yet when the
record is restored from a save. The file is therefore two things at once: a small
fixed-capacity memory with a defined eviction rule, and a machine for resolving dangling
entity references as the level spawns.

A hit record is what tells the brain "something hostile is over there and it is shooting at
me" even when nothing has been seen or heard. Because of that, the filtering rules at the
recording end — who is ignored, what a zero-magnitude hit means, what happens to a friendly
hit — are the behaviour, not bookkeeping.

## State

```text
RECORD HitMemory
  creature        : Creature               # the owner; always present
  stalker         : optional<Stalker>      # the same creature, when it is a stalker
  hits            : optional<list<HitRecord>>   # NOT OWNED; may be the group's shared list
  delayed         : list<DelayedHit>       # loaded records whose attacker has not spawned
  max_hit_count   : int                    # capacity of `hits`, from configuration
  last_hit_object : int (16-bit)           # most recent attacker; none = all-ones
  last_hit_time   : int                    # when, on the level clock

RECORD DelayedHit
  object_id : int (16-bit)    # the attacker to wait for
  record    : HitRecord       # everything except the resolved attacker
```

A hit record (defined in the shared memory space) carries: the attacker, the attacker's
position and navigation vertex at the moment of the hit, the creature's own position and
vertex, the direction of the blow in world space, the bone that was struck, the magnitude,
the squad mask of which group members know about it, a level timestamp, and an enabled flag.

**Invariants** — the list is *not owned*. It may be this creature's own or the whole group's,
and the manager must never free it. Everything else about the class follows from that.

The capacity defaults to **one**. A creature with default tuning remembers exactly one
attacker, and recording a second evicts the first. That is a deliberate coarseness: the
brain reacts to *the* threat, not to a threat list, and the tuning key exists so that a
squad-shared list can be made long enough to hold one entry per member.

## Recording a hit

### `add(amount, local_direction, who, bone)`

**Contract** — the main entry point, called from the damage path. Decides whether this hit
becomes a memory at all, and if so whether it creates a record or updates one.

```text
FUNCTION add(amount, local_direction, attacker, bone)
  IF debug flag "ignore the player" is set AND attacker is the player THEN RETURN
  IF this creature is dead THEN RETURN          # the dead do not learn
  IF attacker is this creature THEN RETURN      # self-damage is not an attack

  IF attacker exists AND amount is non-zero THEN
    last_hit_object = attacker.id
    last_hit_time   = now                       # recorded BEFORE any friend/foe filtering:
                                                # "who hurt me last" is true even for a
                                                # friendly shot or an unrecorded one
  fire the script hit callback(self, amount, local_direction, attacker, bone)

  world_direction = this creature's transform applied to local_direction

  IF attacker is not a living entity THEN RETURN          # a rock, an anomaly: no memory
  IF this creature considers the attacker a friend THEN RETURN

  existing = the record for this attacker, if any
  IF none THEN
    record = new record filled from (attacker, self, squad mask)
    record.amount = amount
    IF the list is full THEN
      replace the record with the OLDEST level timestamp
    ELSE
      append
  ELSE
    refill the existing record from (attacker, self, mask | existing mask)
    record.amount = max(amount, record.amount)     # remember the WORST blow, not the last
```

**Invariants** — several, all behavioural:

- The last-attacker pair is set *before* the friend test, so a creature knows it was shot by
  a team-mate even though it will not form a hostile memory. Scripts read that pair to
  implement team-kill reactions.
- The script callback fires for *every* hit, including friendly ones and including
  zero-magnitude ones. It is the game layer's only notification that a hit happened.
- A friendly hit forms **no memory**. Friendly fire does not turn a squad on itself.
- An update keeps the **maximum** magnitude seen, not the most recent. The brain's reaction
  is keyed on how badly this attacker has hurt it in total, so a grazing follow-up shot must
  not downgrade a serious wound.
- The refilled record refreshes the attacker's *position*, so a repeated attacker's
  remembered location tracks them. That is the mechanism by which sustained fire keeps a
  creature's aim on the shooter without needing line of sight.
- Eviction is oldest-first by level timestamp, not lowest-magnitude. With a capacity of one
  that is the same thing; with a squad list it means a squad remembers the most recent
  attackers.

**Notes** — the direction is transformed into world space and then, on this path, not used:
it is the record-filling call that captures the direction. A rebuild need not compute it
separately.

The squad mask is all-ones for a non-stalker and the stalker's own member bit otherwise. All
ones means "everybody knows", which for a creature with no squad is the only meaningful
answer.

### `add(attacker)`

**Contract** — records a hit of zero magnitude from straight ahead. Used to make a creature
aware of an attacker without hurting it — a near miss, a scripted alert.

**Notes** — because the magnitude is zero, the last-attacker pair is *not* updated by this
path, but a memory record is still formed. The asymmetry is deliberate and worth preserving:
a near miss makes you aware of someone, it does not count as having been hit by them.

### `add(record)`

**Contract** — inserts an already-built record, used by the save-load path and by squad
sharing. Same capacity and eviction rules. On merging with an existing record it takes the
incoming record wholesale but **unions the squad masks**, so knowledge accumulated by other
squad members is not lost.

**Notes** — unlike the main path this one unconditionally reads the stalker's squad mask
without checking whether there is a stalker. Every caller happens to be a stalker path; a
rebuild should guard it.

## `update`

**Contract** — runs once per perception cycle. Two jobs:

```text
FUNCTION update()
  clear_delayed_objects()        # give up waiting for anything not yet spawned
  remove every record whose attacker is gone:
    - the attacker no longer exists
    - the attacker is marked for destruction
    - the attacker has acquired a PARENT
```

**Invariants** — the third condition is the non-obvious one. An entity that has gained a
parent has been picked up, mounted or absorbed — it is now inventory or a passenger, not a
free actor in the world — and a memory of being hit by it is meaningless. Dropping those
records is what stops a creature from hunting a rifle lying in someone's hands.

**Notes** — the delayed list is cleared at the *start* of every update, which means a loaded
record whose attacker has not spawned by the first perception cycle after the load is
discarded. That is a one-frame window, and it is why the load path resolves what it can
immediately rather than relying on the deferred mechanism alone.

## Persistence

### `save`

**Contract** — writes nothing at all if the creature is dead. Otherwise a count byte followed
by one entry per record: the attacker's identifier, the attacker's remembered vertex and
position, the creature's own remembered vertex and position, the timestamps as **ages**, the
direction, the bone and the magnitude.

**Invariants** — timestamps are stored as *elapsed time since the hit*, not as absolute
level times, and are clamped at zero. The level clock restarts on load, so an absolute
timestamp would be meaningless; an age survives the restart and is converted back on load.

The count is a single byte, capping a saved hit memory at 255 records regardless of the
configured capacity.

**Notes** — a dead creature saves nothing, and the load path likewise reads nothing for a
dead creature, so the two stay in step. A rebuild that writes an empty count for the dead
instead would be cleaner and equivalent.

### `load`

**Contract** — reads the records back, converts each age into a level time, and then resolves
each attacker:

```text
FUNCTION load(packet)
  IF this creature is dead THEN RETURN
  count = packet.read_byte()
  REPEAT count TIMES
    delayed.object_id = packet.read_entity_id()
    record = read the rest of the fields
    record.attacker = look up object_id among the live objects
    FOR EACH stored age: record.time = now - age

    IF record.attacker was found THEN
      add(record)                      # resolved immediately
      CONTINUE

    remember it in the delayed list
    IF no spawn callback is already registered for (object_id, this creature) THEN
      register one that will call back when that object spawns
```

**Invariants** — the deferred path exists because a save restores entities in an arbitrary
order and a creature may be restored before its attacker. The registration is keyed on the
pair (awaited object, waiting creature), so two creatures waiting for the same attacker each
get their own callback and a creature never registers twice for the same attacker.

The callback is not registered on a dedicated server, which has no client-side spawn
machinery; there, an unresolved record is simply lost.

### `on_requested_spawn`

**Contract** — the awaited entity has appeared. Finds its delayed record, attaches the now-live
attacker, inserts the record — **but only if this creature is still alive** — and removes the
delayed entry either way.

**Invariants** — the delayed entry is removed whether or not the record is inserted, and the
scan stops at the first match. A creature that died between the load and the spawn quietly
drops the memory, which is correct: the dead do not remember.

### `clear_delayed_objects`

**Contract** — deregisters every outstanding spawn callback this creature installed and empties
the delayed list. Called at the start of every update and at destruction.

**Invariants** — the deregistration must happen before the creature is destroyed or the spawn
manager will later call into freed memory. That is why it is in the destructor as well as the
update.

## Queries and editing

### `hit(attacker)`

**Contract** — the record for a given attacker, or nothing. Matching is by entity identifier,
with a deliberate special case: asking for *no* attacker matches a record that has no
attacker, rather than matching nothing.

### `enable`

**Contract** — marks one record enabled or disabled without removing it. A disabled record is
kept but ignored by the brain, which is how a script suppresses a reaction without erasing
the creature's knowledge.

### `remove`

**Contract** — removes one record, identified by its storage location rather than by its
contents. The caller is iterating and holds a handle to the entry.

**Notes** — identity-by-address is an artifact of how the caller holds the entry. A rebuild
with stable record identifiers should use one; what must survive is that the removal targets
*that* entry and not merely the first one matching its attacker, since a shared squad list
can hold several.

### `remove_links`

**Contract** — called when an object is being destroyed anywhere in the world. Clears the
last-attacker pair if it named that object, and removes that object's record.

**Invariants** — every holder of a reference to a game object gets this call, and missing it
leaves a dangling reference that the next perception cycle will follow. The last-attacker
identifier is cleared to the all-ones sentinel and the time to zero together; leaving a stale
time with a cleared identifier would let a script read "hit by nobody, two seconds ago".

## `reinit` and `reload`

**Contract** — reinitialization clears the list binding to none and resets the last-attacker
pair. It does **not** clear the list contents, because the list may belong to a group.

Reload reads the capacity from the creature's configuration section, defaulting to one when
the key is absent.

**Invariants** — reinitialization unbinding rather than clearing is the correct behaviour for
a shared list and the dangerous behaviour for a private one: a creature reinitialized while
holding its own list leaves that list's contents behind for whoever rebinds it. The owning
memory manager is responsible for the private list's contents.

## `Load`

**Contract** — deliberately empty. The tuning that could live here lives in the reload path,
which is called at a point where the configuration section is known. The method exists to
satisfy the shape every sense manager has.
