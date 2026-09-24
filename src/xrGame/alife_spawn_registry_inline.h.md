# src/xrGame/alife_spawn_registry_inline.h

> Artefact placement inside an anomaly, and two small helpers the spawn walk depends on.

**Needs** — [`alife_spawn_registry.h`](alife_spawn_registry.h.md)
**Used by** — [`alife_spawn_registry.h`](alife_spawn_registry.h.md)
**Tier floor** — T2: an indexed lookup into authored position data

## Purpose

Three of these are accessors; one is a real decision that has nowhere else to live.
Substance of the registry is in
[`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md).

## `assign_artefact_position`

**Contract** — places an object at a position drawn from the anomaly's own authored
candidate set. Fails hard if the anomaly declares no candidates.

```text
FUNCTION assign_artefact_position(anomaly, object)
  object.game_vertex = anomaly.game_vertex
  REQUIRE anomaly.artefact_spawn_count > 0
    ELSE FAIL WITH "anomaly is outside the navigation map but is used for artefact generation"
  index = anomaly.artefact_position_offset + anomaly.random_below(anomaly.artefact_spawn_count)
  object.position     = artefact_spawn_positions[index].level_point
  object.level_vertex = artefact_spawn_positions[index].level_vertex
  object.distance     = artefact_spawn_positions[index].distance
```

**Invariants** — this is the mechanism the whole artefact economy rests on, and its shape
is worth stating plainly.

- The candidate positions are **one flat array for the entire spawn file**, and each
  anomaly owns a contiguous *slice* of it, identified by an offset and a count stored on
  the anomaly. The array and the slices are computed when the spawn file is built, by
  sampling the navigation mesh inside each anomaly's volume. A rebuild must either
  reproduce that offline computation or generate candidates at load time from the
  anomaly's shape — and if it does the latter, the offsets in the shipped data become
  meaningless and every anomaly's artefact positions change.
- A count of zero means the anomaly's volume contains no navigation-mesh positions — it
  was authored outside the walkable map — and using it for artefact generation is a data
  error rather than a runtime condition. The message says so.
- The draw uses the **anomaly's own** random stream, not the registry's. So the same
  anomaly produces a reproducible sequence of positions independent of what else the
  world is doing, which is what makes an anomaly's artefact placement feel like a property
  of the anomaly.
- The object's coarse graph vertex is taken from the anomaly before the position is
  chosen, because every candidate lies within that vertex by construction.

## `process_spawns`

**Contract** — sorts a list of spawn identifiers and removes duplicates, in place. Used to
normalize both the input and the output of the spawn walk, whose membership tests are
binary searches.

## `spawn_id`

**Contract** — resolves an authored spawn-story identifier to a spawn identifier. Absence
is a fault — a script naming a spawn story that does not exist is a script error, not a
condition.

## `header` and `spawns`

**Contract** — read access to the spawn file's header and to the spawn graph.
