# src/xrGame/alife_smart_terrain_task.cpp

> A destination, stated either as a named patrol point or as a pair of navigation vertices, and resolved to the triple an offline mover needs.

**Needs** — [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path_storage.h`](../xrAICore/Navigation/PatrolPath/patrol_path_storage.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_point.h`](../xrAICore/Navigation/PatrolPath/patrol_point.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md)
**Used by** — reached through its declarations in [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md); callers name that, not this file.
**Tier floor** — T2: a two-case resolution over shared navigation data

## Purpose

A smart terrain hands out jobs, and a job needs a *place*. This is that place. It is a
small value passed by copy from the smart terrain's scripts down to the offline mover, and
its whole content is that a place can be authored in two ways which must produce the same
three answers.

**By patrol point** — the authoring form. A designer names a patrol path and a point on
it; the task holds a reference to that point and reads its graph vertex, level vertex and
position straight out of the level data.

**By vertex pair** — the computed form. A script that has worked out a coarse vertex and a
fine vertex, and wants the position derived from them.

The two forms are distinguished by which fields are set, not by a tag, and every accessor
branches on that. A rebuild should make it an explicit choice of two shapes.

## State

```text
RECORD SmartTerrainTask
  patrol_point   : optional<ref PatrolPoint>   # set in the authored form
  game_vertex_id : optional<int>               # set in the computed form
  level_vertex_id: optional<int>               # set in the computed form
```

Invariant: exactly one form is populated. The authored form leaves both vertex fields at
their invalid values and holds a point reference; the computed form holds the two vertices
and no point. The accessors test the *vertex* fields to decide which form they are in.

The reference into the patrol point is a pointer into the level's shared, immutable patrol
data. It is valid only while that level's patrol paths are loaded — a task is a
short-lived value handed from a smart terrain to a mover within one simulation tick, not
something to store across a level change. A rebuild storing a path name and index instead
removes the lifetime hazard at the cost of a lookup per read.

## `setup_patrol_point`

**Contract** — resolves a patrol path by name and a point index within it, and binds the
task to that point. Fails hard if the path does not exist or the index is out of range, and
if the task is already bound.

```text
FUNCTION setup_patrol_point(path_name, point_index)
  REQUIRE not already bound
  path = patrol_path_storage.lookup(path_name)     # FAIL WITH unknown patrol path
  patrol_point = path.point(point_index).data
```

**Notes** — the file carries a compiled-out fallback for the case where the named path does
not exist, intended for modded data that references a path the level does not contain. It
tries the same name with its trailing number decremented by one, registers that path under
the missing name as an alias, and retries; if that fails it repeats the lookup with the
failure made fatal, so a genuinely missing path still stops the game. The heuristic
assumes the naming convention the shipped data uses (a base name plus an index suffix) and
is disabled in the shipping build. A rebuild should not implement it: a missing patrol path
is bad data, and the fallback merely moves the creature somewhere the designer did not
choose.

## `game_vertex_id`

**Contract** — the coarse navigation vertex of the destination. Never fails; asserts that
whichever source it uses yields a valid vertex.

```text
FUNCTION game_vertex_id() -> int
  IF game_vertex_id is unset
    RETURN patrol_point.game_vertex_id
  RETURN game_vertex_id
```

## `level_vertex_id`

**Contract** — the fine navigation vertex of the destination, from the same two sources.

```text
FUNCTION level_vertex_id() -> int
  IF level_vertex_id is unset
    RETURN patrol_point.level_vertex_id
  RETURN level_vertex_id
```

**Notes** — the validity check on this path asserts the *game* vertex rather than the
level vertex it is about to return, in both branches. A rebuild should check the value it
returns.

## `position`

**Contract** — the precise world position of the destination, and the only accessor whose
two branches genuinely differ.

```text
FUNCTION position() -> vector
  IF level_vertex_id is unset
    RETURN patrol_point.position            # authored: the designer's exact point

  IF the game vertex's level is the level currently loaded
    RETURN level_graph.position_of(level_vertex_id)   # fine: the navigation vertex's centre
  RETURN game_graph.vertex(game_vertex_id).level_point  # coarse: the graph vertex's own point
```

**Invariants** — this is the load-bearing branch of the file. A destination on **another
level** has no usable fine position, because that level's navigation mesh is not loaded;
the game graph's own per-vertex point is the best available answer and is precise enough
for offline travel, which only ever compares vertices. A destination on **the loaded
level** must use the fine mesh, because the position will be handed to a creature that is
about to be promoted online and must land on walkable ground.

Note the asymmetry between the three accessors: the two vertex accessors branch on
*whether the field is set*, while the position accessor branches on *which level the
destination is on*. Both distinctions are necessary and neither subsumes the other.

The authored form sidesteps the question entirely: a patrol point carries an exact
position authored into the level, and that position is right whether or not the level is
loaded, because it was recorded rather than derived.
