# src/xrGame/sound_memory_manager.cpp

> A creature's ear: it weighs an incoming sound by category, compares it against a threshold that rises with every noise and decays back down between them, and records what survives into a bounded, squad-shared memory that persists across saves.

**Needs** — [`sound_memory_manager.h`](sound_memory_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`client_spawn_manager.h`](client_spawn_manager.h.md) · [`sound_user_data_visitor.h`](sound_user_data_visitor.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`sound_memory_manager.h`](sound_memory_manager.h.md)
**Tier floor** — T2: event-rate work with a frozen save encoding

## Purpose

Hearing in this engine is *event-driven*: the audio layer tells a creature that a sound
arrived, with the emitter, a category bitset, a world position and a power. This file
turns that event stream into a memory a brain can reason over, and the interesting part is
everything it decides **not** to record.

Three mechanisms do that work, and each is a separate idea:

1. **Category weighting.** Raw acoustic power is not what a creature reacts to. A gunshot
   is multiplied by one factor, an item noise by another, a creature noise, an anomaly and
   the world by three more. All five come from configuration, so a species can be tuned to
   care about gunfire and ignore wind.
2. **An adaptive threshold.** The bar a sound must clear is raised to the power of the
   last sound heard and decays exponentially back to a configured floor. A creature that
   has just heard a firefight does not notice a footstep; the same creature after ten
   quiet seconds does.
3. **Redundancy filters.** A creature does not remember its own noises, its own
   equipment's noises, or the noise of something it can currently *see* — vision already
   knows about that, and a sound record would be a worse duplicate of a better fact.

## State

The records are in [`sound_memory_manager.h`](sound_memory_manager.h.md). Two constants
and one configuration block belong here:

```text
# configuration, read from the creature's section (defaults in parentheses)
  DynamicSoundsCount    -> max_sound_count       (1)
  sound_threshold       -> min_sound_threshold   (0.05)
  self_sound_factor     -> self_sound_factor     (0.0)
  self_decrease_quant   -> sound_decrease_quant  (250 ms)
  self_decrease_factor  -> decrease_factor       (0.95)

# per-category weights, read from `sound_perceive_section` if the creature names one,
# otherwise from its own section — so a family of creatures can share one ear profile
  weapon (10.0) · item (1.0) · npc (1.0) · anomaly (1.0) · world (1.0)
```

**The weapon weight of 10 is the load-bearing number of the file.** It is an order of
magnitude above every other category, which is what makes gunfire the thing that pulls a
creature's attention across a level while ambient noise never does.

**The default capacity of one is also load-bearing** and easy to misread: unless its
configuration says otherwise, a creature remembers exactly *one* sound. The memory is a
"what is the most recent significant noise" register, not a log. Species that need more
raise it in data.

## `feel_sound_new`

**Contract** — the hearing event. Takes the emitter (which may be absent, meaning the
world itself), a category bitset, an optional AI payload, the world position and the raw
power. Always notifies the creature's script-visible sound callback. Records nothing if
the creature is dead or has no record list. Otherwise weights the power, tests it against
the current threshold, possibly records it, and finally raises the threshold to at least
the weighted power. Runs on the simulation thread at event rate — potentially many times
per frame in a firefight — so everything in it must be cheap.

```text
FUNCTION feel_sound_new(emitter, sound_type, payload, position, power)
  IF debug flag "ignore the player" IS set AND emitter IS the player
    RETURN
  IF sounds IS none                      RETURN
  IF payload EXISTS
    payload.accept(visitor)              # lets the brain read speech content, etc.

  object.sound_callback(emitter, sound_type, position, power)   # script sees ALL sounds
  decay_threshold()                      # decay is applied at event time, not per frame
  IF object IS dead                      RETURN                 # a corpse still reports
                                                                # to scripts, remembers nothing

  FOR EACH (category, factor) IN {weapon, item, npc, anomaly, world}
    IF sound_type CONTAINS category
      power = power * factor

  IF power >= sound_threshold
    IF sound_type CONTAINS weapon_shooting
      IF emitter IS a living entity AND emitter IS NOT self AND emitter.team != my team
        hit_memory.add(emitter)          # see Notes: treat being shot at as being hit
    record(emitter, sound_type, position, power)

  last_sound_time = now
  sound_threshold = max(sound_threshold, power)
```

**Invariants** — the script callback fires for *every* sound, before any filtering and
regardless of whether the creature is alive. Scripts therefore see the raw event stream
and the brain sees the filtered one. A rebuild that moves the callback behind the
threshold test will break shipped scripts that count shots or react to specific noises.

`last_sound_time` is updated even when nothing was recorded, because it is the decay
clock, not the memory clock.

**Notes** — the shooting case is marked in the source as a fake, and it is: hearing a
hostile weapon fire injects an entry into the *hit* memory, as though the creature had
been shot at. That is how a character under fire from an unseen shooter acquires a target
to react to. The condition is deliberately narrow — a living, hostile, non-self emitter —
so friendly fire and one's own weapon do not produce phantom threats.

The final line is the adaptive part: a loud sound leaves the threshold at its own level,
so the next sound must be at least as loud to register until decay brings the bar down.

## `decay_threshold` (`update_sound_threshold`)

**Contract** — lowers the threshold toward its floor according to how long it has been
since the last sound event. Pure computation on the manager's own fields; called at the
top of every hearing event, never on a timer.

```text
FUNCTION decay_threshold()
  elapsed_quanta = (now - last_sound_time) / sound_decrease_quant
  threshold = max( self_sound_factor * threshold * decrease_factor ^ elapsed_quanta,
                   min_sound_threshold )
```

**Notes** — the exponential is expressed as `exp(quanta * ln(factor))` in the source
rather than as a power, which is an arithmetic detail; what matters is that decay is
**geometric per 250 ms quantum**, defaulting to a factor of 0.95, so the threshold halves
roughly every three and a half seconds of silence and is floored at the configured
minimum.

`self_sound_factor` defaults to **zero**, which multiplies the decayed term away entirely
and makes the threshold snap back to its floor on the first event after any silence. That
is the shipped behaviour for most creatures; species that set it non-zero get a genuinely
sluggish, momentum-carrying ear. The parameter's name suggests it was meant to scale a
creature's tolerance for its *own* noise, and nothing else in the file supports that
reading — treat the name as vestigial and the arithmetic as the contract.

Evaluating decay lazily at event time rather than per frame is correct and cheap: the
threshold has no observable effect except when a sound is being tested against it.

## `record` (the `add` overload taking an emitter)

**Contract** — decides whether a heard sound becomes a memory record, and either refreshes
the existing record for that emitter or creates a new one. Silently drops sounds that fail
a redundancy filter. Requires the emitter, if present, to be a game object; a non-game
object is dropped.

```text
FUNCTION record(emitter, sound_type, position, power)
  IF emitter IS self                            RETURN   # my own noise
  IF emitter.parent IS self                     RETURN   # my own equipment's noise
  IF vision.visible_now(emitter)                RETURN   # I can see it; vision is better
  IF emitter EXISTS AND emitter IS NOT a game object   RETURN

  contributing_mask = stalker ? my squad-member bit : all bits
  existing = record in sounds WHERE record.object IS emitter
  IF existing EXISTS
    refresh existing WITH (sound_type, power, my position, now)
    existing.squad_mask = existing.squad_mask OR contributing_mask
    IF emitter IS none
      existing.position = position       # world sounds have no object to locate
  ELSE
    new = sound_object filled FROM (emitter, self, sound_type, power, contributing_mask)
    IF emitter IS none
      new.position = position
    insert(new)
```

**Invariants** — at most one record per emitter. The single record for an absent emitter
is "the world", so every unattributed noise overwrites the previous one's position.

**Notes** — the filter set is assembled from build switches in the original, most of them
permanently on or off. As shipped, the three active exclusions are the ones above, and the
three *disabled* ones are worth naming because the switches record deliberate reversals:
sounds from non-living objects **are** remembered, sounds from friendly characters **are**
remembered, and sounds from friendlies' equipment **are** remembered. Earlier versions
filtered all three; a rebuild should not reintroduce those filters.

The visible-object exclusion is the subtle one. It means a creature watching an enemy
shoot accumulates no sound memory of it at all — the enemy is in vision memory instead.
The consequence is that sound memory is specifically a memory of the *unseen*, which is
exactly what a brain wants to search toward.

The squad mask is a bitset of which squad members contributed, accumulated by `OR` on
refresh. A record heard by three members carries three bits, which lets the squad's
coordination layer tell a corroborated noise from one member's report.

## `insert` (the `add` overload taking a record)

**Contract** — puts a record into the bounded list, optionally skipping if a record for
the same emitter already exists. When the list is at capacity, **the record with the
oldest refresh time is overwritten**; the list never grows past its configured capacity.
Asserts the capacity is non-zero.

```text
FUNCTION insert(record, skip_if_present = false)
  IF skip_if_present AND a record for record.object EXISTS
    RETURN
  IF count(sounds) >= max_sound_count
    OVERWRITE the record with the smallest level_time
  ELSE
    APPEND record
```

**Notes** — eviction is by least-recently-refreshed, not by weakest power or lowest
priority. That is the right choice for an ear: an old loud noise is less actionable than a
recent quiet one, and refreshing a record on every repeat means a continuing source keeps
its slot.

## `update`

**Contract** — per-frame maintenance. Retires deferred spawn requests, drops records whose
emitter has been picked up by something (acquired a parent), and in checked builds
recomputes which record is the most important. Cheap; the list is tiny.

```text
FUNCTION update()
  clear_delayed_objects()
  REMOVE r FROM sounds WHERE r.object EXISTS AND r.object HAS a parent
  # checked builds only: select the record whose priority rank is lowest
```

**Notes** — the parent test is the "somebody picked it up" rule: a weapon lying on the
ground that made a noise is a location worth remembering, but once it is inside an
inventory its position is the carrier's and the record is misleading. The same rule
appears in the vision memory.

## `priority`

**Contract** — ranks a record by consulting the registered type table: the result is the
smallest registered rank among the types whose bits are **all** present in the record's
type bitset. Returns the maximum integer when nothing matches, meaning "least important".

**Notes** — the containment test is subset, not intersection: a registered type matches
only when every one of its bits is set on the record. That lets a compound type such as
"weapon and shooting" be ranked separately from plain "weapon".

## `enable`

**Contract** — finds the record for an entity and sets its enabled flag. Does nothing if
no such record exists. Used to suppress a record's influence on the brain without
forgetting it, so the memory survives to be re-enabled.

## `remove`, `remove_links`

**Contract** — `remove` drops one record identified by address. `remove_links` drops the
record naming a given entity and clears the selected record if it named that entity.
Neither fails on absence.

**Invariants** — `remove_links` is part of the engine-wide rule that a destroyed entity
must be unreferenced everywhere before its memory is released (see the runtime invariants
in the system requirements). Every registry that stores an entity reference implements
this hook; the sound memory's obligation is one record and one selection.

## `save`

**Contract** — writes the creature's whole sound memory into the save stream. **Writes
nothing at all if the creature is dead** — the dead remember nothing, and the loader makes
the matching decision, so this is not an error, it is the format. Fields are written per
record in a fixed order.

```text
FUNCTION save(stream)
  IF object IS dead   RETURN
  WRITE count(sounds)                  : int (8-bit)     # caps a creature at 255 records
  FOR EACH r IN sounds
    WRITE r.object.id OR 0xffff        : int (16-bit)    # 0xffff means "the world"
    WRITE r.object_params.level_vertex : int (32-bit)
    WRITE r.object_params.position     : 3 reals
    WRITE r.self_params.level_vertex   : int (32-bit)
    WRITE r.self_params.position       : 3 reals
    WRITE now - r.level_time           : int (32-bit)    # AGE, not an absolute instant
    WRITE r.sound_type                 : int (32-bit)
    WRITE r.power                      : real
```

**Invariants** — timestamps are stored as **ages relative to the moment of saving**, never
as absolute clock values, and are clamped at zero. This is what lets a memory survive a
load into a session whose world clock starts somewhere else entirely; the loader adds the
age back to its own clock. Every time field in every memory manager uses this convention.

The count is written as a single byte, which silently caps a saved memory at 255 records.
No creature's configured capacity comes near that, so it has never bitten, but a rebuild
that raises capacities must widen the field and version the format.

## `load`

**Contract** — reads the record block back. Reads nothing if the creature is dead,
mirroring `save`. For each record it tries to resolve the emitter identifier against the
live object registry; if the entity exists, or the record is a world sound, the record is
inserted immediately. **If the entity does not exist yet, the record is parked and a
spawn callback is registered**, so that the memory is restored the moment the entity
arrives. Duplicate suppression is on for every insertion here.

```text
FUNCTION load(stream)
  IF object IS dead   RETURN
  count = READ int (8-bit)
  REPEAT count TIMES
    id = READ int (16-bit)
    r  = READ the fields in the order `save` wrote them
    FOR EACH age field: r.time = now - age
    r.object = id IS 0xffff ? none : live object WITH id
    IF r.object EXISTS OR id IS 0xffff
      insert(r, skip_if_present = true)
      CONTINUE
    park (id, r) IN delayed_objects
    IF no spawn callback for (id, me) IS registered AND this is not a dedicated server
      register spawn callback (id, me) -> on_requested_spawn
```

**Notes** — the deferred path exists because a save restores entities in an order this
manager does not control, and because an entity may be *offline* in the alife sense and
not yet promoted into the level. Without it, a creature would silently forget every noise
made by anything that happened to load after it. The dedicated-server exclusion is because
a server without a client has no client-side spawn pipeline to hang the callback on.

Registering the callback only when one is not already present matters: several memory
managers on the same creature — sound, vision, hit — wait on the same entity, and they
share one callback that fans out through the creature's memory facade.

## `on_requested_spawn`

**Contract** — the callback for a parked record's entity arriving. Finds the parked entry
for that entity, attaches the now-live object to the record, inserts it if the creature is
still alive, and removes the parked entry either way. Returns after the first match — a
creature parks at most one record per entity.

## `clear_delayed_objects`

**Contract** — unregisters every outstanding spawn callback this creature owns and drops
the parked records. Called at the top of every update and at teardown.

**Invariants** — this is the counterpart of the registration in `load` and closes the
lifetime hole it opens: a creature that dies, unloads or is destroyed while waiting on a
spawn must not leave a callback pointing at itself. Calling it every frame means a parked
record survives at most one update after the load — a rebuild could keep them longer, but
must then own the unregistration on every teardown path.

## `reinit`, `reload`, `Load`, teardown

**Contract** — `reinit` detaches the record list, empties the priority table, zeroes the
decay clock and resets the threshold to its floor. `reload(section)` reads the
configuration block listed under State, including the indirection through an optional
shared perception section. `Load` does nothing and exists only to complete the lifecycle
shape. Teardown clears the deferred requests and the selection.

**Notes** — `reinit` clearing the priority table is why the owning brain re-registers its
sound-type ranks after every reinitialisation rather than once at construction.
