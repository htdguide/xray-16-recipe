# src/Layers/xrRender/ParticleGroup.cpp

> A particle **group**: the authored timeline that starts and stops several effects at named times, plus the runtime that plays it, spawns per-particle child effects, and reports one merged bounding volume to the visibility walk.

**Needs** — [`ParticleGroup.h`](ParticleGroup.h.md) · [`ParticleEffect.h`](ParticleEffect.h.md) · [`PSLibrary.h`](PSLibrary.h.md) · [`dxParticleCustom.h`](dxParticleCustom.h.md) · [`Include/xrRender/ParticleCustom.h`](../../Include/xrRender/ParticleCustom.h.md) · [`xrParticles/psystem.h`](../../xrParticles/psystem.h.md)
**Used by** — [`ParticleGroup.h`](ParticleGroup.h.md)
**Tier floor** — T1: the group record is read from the shipped particle library as a chunked byte image with frozen identifiers and a hard version gate.

## Purpose

An effect is one emitter. A **group** is what the game actually names: an explosion is a flash effect at t=0, a smoke effect from 0.1 to 4 seconds, and a debris effect that spawns a trailing child from each of its particles. This file holds both halves of that idea — the authored group record, and the runtime that plays it.

The runtime is not a container of effects so much as a small scheduler: it keeps one **item** per authored entry, starts and stops that item as the group's clock crosses the entry's two times, and additionally manages the two kinds of *child* effect an entry may spawn per particle. The group is itself a renderable visual, so the visibility pass sees one object where the frame sees many.

## State

```text
RECORD GroupEntry                    # one authored line of the timeline
  flags             : int (32-bit)
      deferred_stop        = bit 0   # at time1, stop emission and let particles drain
      has_on_play_child    = bit 1   # spawn a child that FOLLOWS each live particle
      enabled              = bit 2   # a disabled entry is loaded but never played
      on_play_child_rewind = bit 4   # restart a following child that finished early
      has_on_birth_child   = bit 5   # spawn a free child where each particle is born
      has_on_dead_child    = bit 6   # spawn a free child where each particle dies
  effect_name        : text          # library key of the effect this entry plays
  on_play_child_name : text
  on_birth_child_name: text
  on_dead_child_name : text
  time0              : real          # seconds into the group at which the entry starts
  time1              : real          # ... and at which it stops
```

Bit 3 is unused and must stay unused: the shipped files were written with it clear and a rebuild that reassigns it changes the meaning of existing data.

```text
RECORD GroupDefinition
  name       : text
  flags      : int (32-bit)          # read and written, but nothing consumes any bit
  time_limit : real                  # seconds; zero means "no limit of its own"
  entries    : list<GroupEntry>
```

```text
RECORD GroupItem                     # the runtime counterpart of one entry
  effect            : ParticleEffect or none   # the entry's own emitter
  children_related  : list<ParticleEffect>     # one per live particle, index-parallel to them
  children_free     : list<ParticleEffect>     # fire-and-forget, deleted when they finish
```

```text
RECORD ParticleGroup                 # is-a renderable visual, is-a particle-custom visual
  definition      : GroupDefinition or none
  current_time    : real             # seconds since play, the timeline's clock
  initial_position: vector
  items           : list<GroupItem>  # exactly parallel to definition.entries
  runtime_flags   : int (8-bit)      # playing = bit 0, deferred_stop = bit 1
```

**Invariants**

- `items` is index-parallel to `definition.entries` at all times. Every access pairs an entry with its item by index, and the code asserts the lengths match.
- `children_related` is index-parallel to the item effect's **live particle array**. The birth hook appends, the death hook removes by the dying particle's index, and the per-frame update walks both together. This is the strictest invariant in the file: chapter 12's particle removal is a swap-with-last, so the child list must be maintained with the same swap-with-last discipline or every following child snaps to the wrong particle.
- A following ("on play") child is created *before* it is needed by index and removed in the death hook, so it exists for exactly the lifetime of its particle. A stopped following child is not deleted; it is moved to the free list so its remaining particles can drain.
- A free ("on birth" or "on dead") child **must not be a looped effect** — one with no time limit would never finish and the list would grow without bound. The engine treats a looped free child as a fatal authoring error rather than leaking.
- The group's own bounding volume is the union of the boxes of every playing effect and child, recomputed each frame, falling back to a point at the initial position when nothing plays.
- A group stops when its clock passes its time limit (if it has one) *or* when a deferred stop has drained every effect and every child.

## `load` — read a group definition

**Contract** — reads one group out of a chunked reader. The version gate is **hard and logged**: a mismatched version returns failure with a message rather than attempting a partial read, which is how a library from a different game generation is refused. Allocates one entry per authored line.

```text
FUNCTION load(reader) -> bool
  require chunk VERSION; IF version != 3 THEN log and RETURN false
  require chunk NAME;    name = read_string
  read chunk FLAGS into flags
  IF chunk TIME_LIMIT present THEN time_limit = read_real ELSE time_limit = 0

  derive_limit = (time_limit == 0)       # an explicit limit wins over the derived one
  IF chunk EFFECTS present THEN
    FOR count = read_int32 times
      entry.effect_name, on_play_child_name, on_birth_child_name, on_dead_child_name = four strings
      entry.time0, entry.time1 = two reals
      entry.flags = int32
      IF derive_limit THEN time_limit = max(time_limit, entry.time1)
  RETURN true
```

**Notes** — Chunk identifiers are frozen: version 0x0001, name 0x0002, flags 0x0003, entries 0x0004, time limit 0x0005. Identifier 0x0007 is reserved for a second entry-list layout that the current record version does not use — a rebuild must not reuse it.

The derived time limit is the important behaviour here. A group whose file carries no explicit limit gets one equal to the latest `time1` among its entries, so it ends when its last scheduled effect ends. A group that *does* carry a limit keeps it even if an entry runs past — such an entry is cut off mid-play. Both cases appear in shipped data.

The record version is 3; the group format changed twice before shipping and only the third form survives in the game's library.

## `load_from_config`, `save`, `save_to_config`

**Contract** — the same record in the configuration-file form, used for loose per-group files, and the two writers. Entries live in sections named `effect_0000`, `effect_0001`, … with a zero-padded four-digit index; keys are `effect_name`, `on_play_child`, `on_birth_child`, `on_death_child`, `time0`, `time1`, `flags`; the group's own values are in a section named `_group`, keys `flags`, `effects_count`, `timelimit`.

**Notes** — The text form takes the time limit as written and never derives it, unlike the binary form. The key is spelled `on_death_child` while the field it fills is the *dead*-child name — the two spellings are frozen on their respective sides.

The text writer blanks a child name whose flag is clear, so a round trip through the text form loses the names of disabled children. The binary writer keeps them. This is an asymmetry in the shipped tooling, not a rule to reproduce faithfully, but a rebuild that fixes it must accept that files already written have the blanks.

## `compile` — build the runtime from the definition

**Contract** — discards any existing items, then creates one emitter per authored entry through the renderer's model factory and installs the group's own birth and death hooks on it, passing the entry's index as the hook's parameter. Allocates one effect per entry. Idempotent: compiling again tears down first.

```text
FUNCTION compile(definition)
  FOR EACH item IN items: item.clear()          # returns every effect and child to the model pool
  items = empty
  IF definition is none THEN RETURN
  items = one item per entry
  FOR EACH entry, index IN definition.entries
    effect = renderer.create_particle_effect(entry.effect_name)
    effect.set_birth_death_callbacks(group_birth, group_death, owner = self, param = index)
    items[index].effect = effect
```

**Notes** — Effects are obtained from and returned to the renderer's model pool rather than constructed directly, because a group's effects are created and destroyed constantly during play and each one carries a compiled action list worth sharing.

## `on_frame` — run the timeline

**Contract** — advances the group clock, starts and stops items as it crosses each entry's two times, pumps every item, merges the bounding volumes, and retires the group when a deferred stop has drained. Holds the group's own lock for the whole call, because the game may query or re-place the group from another thread.

```text
FUNCTION on_frame(frame_ms)
  LOCK group DURING
    IF definition is none OR NOT playing THEN
      visibility volume = point at initial_position; RETURN

    was = current_time
    dt  = frame_ms in seconds
    FOR EACH entry, item IN pairs
      IF NOT entry.enabled THEN CONTINUE
      IF item.is_playing() THEN
        IF entry.time1 IS CROSSED BY [was, was + dt] THEN item.stop(entry.deferred_stop)
      ELSE IF NOT deferred_stop THEN
        IF entry.time0 IS CROSSED BY [was, was + dt] THEN item.play()
    current_time = was + dt

    IF time_limit > 0 AND current_time > time_limit AND NOT deferred_stop THEN stop(deferred = true)

    playing_any = false
    box = empty
    FOR EACH entry, item IN pairs
      item.on_frame(frame_ms, entry, box, playing_any)
    IF deferred_stop AND NOT playing_any THEN clear playing and deferred_stop
    IF box is non-empty THEN visibility volume = box and its sphere
```

**Invariants**

- A start or stop fires only on the frame in which the clock **crosses** the time, tested as "the time lies in the half-open interval this frame covers". A time that falls exactly on a frame boundary fires once. An entry whose `time0` is passed while the group is already stopping is never started, which is what stops an explosion from spawning its late smoke after the effect was cancelled.
- The clock is advanced in real seconds from the frame's elapsed time, *not* in the particle layer's fixed 33 ms steps. The timeline therefore runs at wall-clock rate while the particles inside it run at a fixed rate with a three-step cap — under a heavy hitch the timeline moves on while the particles fall behind.

## `GroupItem.on_frame` — pump one entry and its children

**Contract** — advances the entry's emitter and all of its children, keeps the following children glued to their particles, restarts those that the rewind flag asks for, retires finished free children, and contributes every live bounding box to the group's union.

```text
FUNCTION item.on_frame(frame_ms, entry, INOUT box, INOUT playing_any)
  IF effect exists THEN
    effect.on_frame(frame_ms)
    IF effect.is_playing() THEN
      playing_any = true; merge effect's box into box
      IF entry.has_on_play_child AND the name is non-empty THEN
        particles = simulator.particles(effect)
        ASSERT count(particles) == count(children_related)
        FOR EACH particle, child IN pairs
          velocity = (particle.pos - particle.prev_pos) / fixed_step_seconds
          child.update_parent(translation to particle.pos, velocity, transform_mode = false)

  FOR EACH child IN children_related
    child.on_frame(frame_ms)
    IF child.is_playing() THEN
      playing_any = true; merge child's box
    ELSE IF entry.on_play_child_rewind THEN
      child.play()                       # the trail restarts under its still-living particle

  FOR EACH child IN children_free
    child.on_frame(frame_ms)
    IF child.is_playing() THEN
      playing_any = true; merge child's box
    ELSE
      return child to the model pool and drop it from the list
```

**Notes** — The velocity handed to a following child is reconstructed from the particle's current and previous positions divided by the **fixed** step, not the frame's elapsed time. It has to be: those two positions are exactly one fixed step apart, whatever the frame rate did.

A following child is placed by a pure translation — it inherits the particle's position but not the emitter's orientation. A free child, by contrast, is placed by the emitter's full transform when the emitter is in transform mode. The difference is visible in authored content and must be preserved.

## `group_birth`, `group_death` — the per-particle child hooks

**Contract** — installed on every entry's emitter with the entry index as parameter. Birth runs the effect layer's own birth behaviour first (random frame, random playback direction), then spawns whichever children the entry asks for. Death likewise, then removes the particle's following child and spawns a dead-child if one is named.

```text
FUNCTION group_birth(group, entry_index, particle, particle_index)
  base particle-birth behaviour for the entry's effect
  IF entry.has_on_birth_child THEN item.start_free_child(entry.on_birth_child_name, particle)
  IF entry.has_on_play_child  THEN item.start_related_child(entry.on_play_child_name, particle)

FUNCTION group_death(group, entry_index, particle, particle_index)
  base particle-death behaviour
  IF entry.has_on_play_child THEN item.stop_related_child(particle_index)
  IF entry.has_on_dead_child THEN item.start_free_child(entry.on_dead_child_name, particle)
```

**Invariants** — the on-birth child is started **before** the on-play child, so that when an entry asks for both, the free child does not disturb the index alignment the following child depends on. The following child is appended by the birth hook in the same order the simulator appends particles, which is what keeps the two lists parallel without any explicit key.

## `GroupItem.start_related_child`, `start_free_child`, `stop_related_child`

**Contract** — create, retire and place the two kinds of child.

Both starters build the child's placement the same way: velocity from the particle's two positions over the fixed step, and a transform that is identity unless the emitter is in transform mode, in which case it is the emitter's matrix with the particle's transformed position as its origin and the velocity rotated into world space. The child inherits the emitter's weapon-view mode. It is played immediately and then placed in moving mode, so its own particles are left behind in the world rather than dragged.

`stop_related_child` takes the dying particle's index, issues a **deferred** stop on that child, moves it to the free list so its remaining particles drain and it is deleted when empty, and closes the hole in the following list with the last element. That swap-with-last is what keeps the list parallel to the simulator's own swap-with-last removal.

`start_free_child` refuses a looped effect outright. A looped free child would never retire, and since one is created per particle birth or death the list would grow for as long as the group plays.

## `play`, `stop`, `is_playing`, `particle_count`, `set_hud_mode`, `get_hud_mode`, `get_time_limit`, `name`

**Contract** — group-level playback and queries, each fanning out over the items. `play` resets the clock to zero, which is what makes replaying a group restart its timeline rather than resume it. `stop` marks deferred or immediate and passes the choice down; an immediate stop also deletes every child, a deferred one leaves them to drain. `particle_count` sums every effect and child. `get_time_limit` returns the group's limit, so a group with a zero limit reads as *not* looped even though it never ends on its own — unlike an effect, where a negative limit means looped.

Every one of these takes the group's lock, because the game calls them from outside the frame.

## `update_parent`, `on_device_create`, `on_device_destroy`, `copy`

**Contract** — placement and device-reset handling fan out to every item and child. `copy` fails fatally for the same reason it does for a single effect: the simulator handles beneath it are not duplicable.
