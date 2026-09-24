# src/xrGame/memory_space.h

> The record shapes of a creature's memory: what it remembers having seen, heard and been hit by, and how each memory is scoped to a squad.

**Needs** — [`xrServerEntities/ai_sounds.h`](../xrServerEntities/ai_sounds.h.md) · [`xrServerEntities/xrServer_Space.h`](../xrServerEntities/xrServer_Space.h.md) · [`xrServerEntities/alife_space.h`](../xrServerEntities/alife_space.h.md) · [`memory_space_impl.h`](memory_space_impl.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_memory_manager.cpp`](agent_memory_manager.cpp.md) · [`agent_memory_manager.h`](agent_memory_manager.h.md) · [`danger_location.h`](danger_location.h.md) · [`danger_manager.cpp`](danger_manager.cpp.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`member_enemy.h`](member_enemy.h.md) · [`memory_manager.cpp`](memory_manager.cpp.md) · [`memory_manager.h`](memory_manager.h.md) · [`memory_space_impl.h`](memory_space_impl.h.md) · [`memory_space_script.cpp`](memory_space_script.cpp.md) · [`script_game_object2.cpp`](script_game_object2.cpp.md) · [`script_game_object3.cpp`](script_game_object3.cpp.md) · _and 6 more_
**Tier floor** — T2: record layouts a creature carries thousands of copies of

## Purpose

A creature does not react to the world; it reacts to *what it remembers of the world*. This
header defines the records that memory is made of. It has no implementation file — the
records are the substance, their filling is in
[`memory_space_impl.h`](memory_space_impl.h.md), and the machinery that maintains them is in
[`memory_manager.cpp`](memory_manager.cpp.md) and the four per-sense managers.

Three things here are load-bearing and one is a compile-time switch that a rebuild should
simply resolve.

**A memory is a snapshot, not a reference.** Every memory record stores the remembered
object's position and navigation vertex *at the time of the memory*, **and the remembering
creature's own position and vertex at that time**. The second is what makes "I saw him from
over there" expressible: a creature can reason about a memory's geometry without having been
told to retain it, and can decide a memory is stale because it has moved, not because the
target has. The object handle is kept too, but a memory outlives the object's relevance.

**A memory is shared across a squad through a bit mask.** Each record carries a mask of
which squad members the memory is valid for. One creature seeing an enemy makes the sighting
available to its whole squad, and one squad member losing sight clears only its own bit.
This is how squad coordination happens without a squad-level blackboard: the memory lives on
each creature and the mask says who it counts for. The mask is 64 bits wide, which caps a
squad at 64 members — a limit nothing checks, and the shipped data never approaches.

**Visibility is a second, independent mask.** A visual memory distinguishes *I remember him*
from *I can see him right now*, per squad member, because the two questions have different
answers for every member of a squad looking from different places.

## State

```text
RECORD ObjectParams                 # a geometric snapshot of one object
  level_vertex_id : int             # the navigation vertex; the invalid marker when unknown
  position        : (real, real, real)   # the object's footprint x and z, its centre's y

RECORD MemoryObject                 # what every memory has
  level_time      : int             # the real-time clock when this memory was last refreshed
  last_level_time : int             # the value level_time had before that refresh
  enabled         : bool            # false suppresses the memory without forgetting it
  object          : GameObject      # who or what is remembered
  object_params   : ObjectParams    # where it was
  self_params     : ObjectParams    # where I was when I formed this memory
  squad_mask      : int (64-bit, one bit per squad member)

RECORD VisibleObject EXTENDS MemoryObject
  visible : int (64-bit)            # per squad member: can they see it *now*

RECORD SoundObject EXTENDS MemoryObject
  sound_type : SoundType            # the perception attributes the sound carried
  power      : real                 # as heard, after attenuation

RECORD HitObject EXTENDS MemoryObject
  direction  : (real, real, real)   # where the damage came from
  bone_index : int                  # which part of me was struck
  amount     : real

RECORD MemoryInfo EXTENDS VisibleObject   # one object, merged across all three senses
  visual_info : bool
  sound_info  : bool
  hit_info    : bool

RECORD NotYetVisibleObject          # a sighting in progress
  object     : GameObject
  value      : real                 # accumulated recognition, 0 to 1
  update_time: int
  prev_time  : int
```

**Invariants**
- A memory's squad mask starts as *all ones* and is narrowed. A newly formed memory counts
  for everybody until somebody's bit is cleared.
- `last_level_time` always holds the previous value of `level_time`. Together they answer
  "how long has this memory been continuously refreshed", which is how a creature tells a
  glimpse from a sustained sighting.
- An object with the invalid navigation-vertex marker is remembered by position only; the
  pathfinder cannot be asked to approach it.
- `MemoryInfo` is a *merge*: one record per remembered object with three flags saying which
  senses contributed. It is what the script layer and the danger system read, not the
  per-sense lists.

## The time switch

The record carries a real-time stamp, a game-time stamp, a first-seen stamp, a last-seen
stamp and an update counter — **but only two of the five are compiled in**, selected by a
switch at the top of the file: the real-time stamp and its previous value. The rest are
disabled.

This matters for two reasons. First, a rebuild should simply carry the two that are used
and drop the others; the record is instantiated per remembered object per creature and its
size is the reason the others were turned off. Second, **the enabled clock is the real-time
one, which does not advance while the game is paused and is not the clock the world's events
are dated by**. Memory decay is therefore measured in wall-clock milliseconds of gameplay,
not in in-game hours — a creature that has not seen you for an in-game day but was offline
for it has not forgotten you.

A second switch, `USE_ORIENTATION`, would add the remembered object's facing to the
snapshot. It is off, and the script binding exports the rotation type anyway, so a script
can hold a rotation but never receives one from a memory.

A third, `USE_STALKER_VISION_FOR_MONSTERS`, is on and makes non-human creatures use the same
sight model as humans rather than a simplified one.

## `SLevelTimePredicate`

**Contract** — orders two memories by refresh time, oldest first. The single ordering the
memory lists are kept in, so that forgetting is a prefix operation.

## `script_register`

**Contract** — exports the memory vocabulary to scripts: the parameter snapshot, the base
memory record with whichever timestamps are compiled in, the entity and object flavours, the
hit, visible, sound and merged records, the sighting-in-progress record, and the danger
record with both of its enumerations. Frozen by conformance criterion 10. The bindings
themselves are in [`memory_space_script.cpp`](memory_space_script.cpp.md).
