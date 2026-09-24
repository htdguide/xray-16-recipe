# src/xrGame/alife_surge_manager.cpp

> Repopulation: work out which authored spawn records have no live entity, run the spawn walk over them, and instantiate what comes back.

**Needs** — [`alife_surge_manager.h`](alife_surge_manager.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`alife_simulator_header.h`](alife_simulator_header.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md)
**Used by** — reached through its declarations in [`alife_surge_manager.h`](alife_surge_manager.h.md); callers name that, not this file.
**Tier floor** — T2: a sweep over the object registry plus the spawn walk

## Purpose

The layer is named for the **surge** — the world-wide event the series uses to justify
repopulating the map — but what it actually contains is the general "spawn everything that
should exist and does not". It runs once when a new game starts, populating the world from
the spawn file, and again whenever the simulation decides to repopulate.

Its whole job is to bridge two representations: the spawn walk wants to know which spawn
records already have a live entity, and the only place that is recorded is on each live
entity, as the identifier of the record it came from.

## State

```text
RECORD SurgeManager
  temp_spawns          : list<SpawnId>   # scratch: what the walk decided to spawn
  temp_spawned_objects : list<SpawnId>   # scratch: what already exists
```

Both are members purely to avoid reallocating across calls; a rebuild makes them local.

## `spawn_new_objects`

**Contract** — the entry point. Repopulates the world. Allocates and registers entities;
blocking and potentially long. Requires that a player exists when it finishes.

```text
FUNCTION spawn_new_objects()
  fill_spawned_objects()                       # 1. what already exists
  spawns.fill_new_spawns(temp_spawns,          # 2. what should exist
                         time.game_time(),
                         temp_spawned_objects)
  spawn_new_spawns()                           # 3. instantiate the difference
  REQUIRE a player exists
```

**Invariants** — the three steps are a set difference computed the long way round: the
spawn walk is given the existing set and prunes against it as it descends, rather than
producing everything and subtracting afterwards. That matters because the walk's pruning
is *structural* — a container whose contents still exist is not re-offered at all, so its
other possible contents are never even rolled.

Game time is passed in because a spawn record may be time-gated. Passing the alife clock
rather than the wall clock is what makes the gate meaningful across a save.

The player check at the end is a canary: the player is itself a spawn record, and a
repopulation that lost the player has gone badly wrong in a way worth catching where it
happened.

## `fill_spawned_objects`

**Contract** — collects the spawn-record identifier of every live entity that came from
one. Sweeps the whole object registry.

```text
FUNCTION fill_spawned_objects()
  temp_spawned_objects.clear()
  FOR EACH object IN objects
    IF spawns.graph has a vertex with object.spawn_record
      temp_spawned_objects.append(object.spawn_record)
```

**Invariants** — the vertex-existence test is what filters out **scripted** entities.
An entity spawned from a configuration section rather than from an authored record carries
no spawn-record identifier (see
[`alife_simulator_base.cpp`](alife_simulator_base.cpp.md)), so it never suppresses a
spawn record. A dropped weapon does not stop the crate it came from being refilled; a
crate that is still in the world does.

Several entities may share one spawn-record identifier — every member of an authored group
does — so the resulting list has duplicates. The spawn walk sorts and deduplicates it
before use, which is why this routine does not.

## `spawn_new_spawns`

**Contract** — instantiates each spawn record the walk selected. Each becomes a registered
server object with a freshly allocated identifier.

```text
FUNCTION spawn_new_spawns()
  FOR EACH spawn_id IN temp_spawns
    template = spawns.graph.vertex(spawn_id).record
    REQUIRE template is a dynamic alife object
    create(out object, template, spawn_id)
```

**Invariants** — the record in the spawn file is a **template**, never the entity itself:
`create` clones it through the serialized round trip and leaves the template untouched, so
the same record can be instantiated again on a later repopulation. A rebuild that
registered the template directly would corrupt the spawn file's in-memory image and make
the second repopulation produce a mutated entity.

The diagnostic build times each creation individually, which is how the cost of a level's
initial population was measured; it is worth keeping, because this loop is the longest
single operation in starting a new game.
