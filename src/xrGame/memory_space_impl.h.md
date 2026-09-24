# src/xrGame/memory_space_impl.h

> How a memory record is filled from a live object: which position is captured, which navigation vertex, and what the previous refresh time becomes.

**Needs** — [`memory_space.h`](memory_space.h.md) · [`GameObject.h`](GameObject.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — [`agent_memory_manager.cpp`](agent_memory_manager.cpp.md) · [`hit_memory_manager.cpp`](hit_memory_manager.cpp.md) · [`memory_manager.cpp`](memory_manager.cpp.md) · [`memory_space.h`](memory_space.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md)
**Tier floor** — T2: snapshot capture on a hot path

## Purpose

The filling half of [`memory_space.h`](memory_space.h.md). It is a separate file because the
records are templates over what is remembered — an object or a living entity — and a
template's definition must be visible where it is used; nothing about the split is a
decision.

What *is* a decision is the small number of choices made here about what a snapshot contains,
and they are the reason this page exists at all.

## `fill` (parameter snapshot)

**Contract** — captures one object's geometry into a snapshot. Tolerates no object at all,
in which case the snapshot is the origin with an invalid navigation vertex — the shape a
memory of something that has since been destroyed takes.

```text
FUNCTION fill(object)
  IF object IS none
    level_vertex_id = invalid ; position = origin ; RETURN
  level_vertex_id = object.ai_location.level_vertex_id
  position        = (object.position.x, object.centre.y, object.position.z)
```

**Notes** — the position is **mixed**: the object's own x and z, but the *centre* of its
bounding volume for y. A creature's recorded position is therefore its footprint
horizontally and its chest vertically. That is what makes a memory usable as an aim point
and as a navigation target at the same time — aiming at a creature's feet misses, and
pathing to a creature's chest is off the navigation mesh.

The navigation vertex is taken as the object *reports* it, not recomputed. A commented-out
alternative here would have snapped the remembered position to the vertex's own position
whenever the object had drifted outside its vertex. It was abandoned; the effect would have
been memories quantized to the navigation grid, which reads as creatures aiming at the wrong
place.

## `fill` (memory record)

**Contract** — refreshes a memory. Rolls the refresh time forward, captures both
snapshots — the remembered object's and the remembering creature's — and installs the squad
mask. Always marks the memory enabled.

```text
FUNCTION fill(object, self, mask)
  last_level_time = level_time
  level_time      = real-time clock now
  self.object     = object
  object_params.fill(object)
  self_params.fill(self)
  squad_mask      = mask
  enabled         = true
```

**Invariants** — the previous time is captured before the new one is written, which is what
makes the pair mean "this refresh and the one before it". A creature that has been
continuously watching has the two close together; one that has just re-acquired a target has
them far apart. Several evaluators depend on that difference rather than on the absolute
age.

**Notes** — `self` is the remembering creature and is snapshotted with the same routine as
the target, so a memory always carries both ends of the sighting. That symmetry is the
single most useful property of the record.

Refreshing a memory always re-enables it. Suppression is therefore transient by design: a
disabled memory comes back the moment the sense that formed it fires again.

## `orientation` / `operator==` / `object_id`

**Contract** — `orientation` extracts heading and pitch from an object's transform,
discarding roll; it is used only when the orientation switch in
[`memory_space.h`](memory_space.h.md) is on, which it is not. Equality compares a memory
against an entity identifier, which is how a sense finds the existing memory for an object
it has just perceived. `object_id` reports an object's identifier, or an all-ones sentinel
for no object — so a memory of a destroyed object never matches a live one.
