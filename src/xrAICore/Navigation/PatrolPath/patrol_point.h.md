# src/xrAICore/Navigation/PatrolPath/patrol_point.h

> Declares one authored waypoint of a patrol route — a named position snapped onto the navigation graphs, with a flag word the game interprets.

**Needs** — [`patrol_point.cpp`](patrol_point.cpp.md) · [`patrol_point_inline.h`](patrol_point_inline.h.md) · [`../game_graph_space.h`](../game_graph_space.h.md) · [`../../../Common/object_interfaces.h`](../../../Common/object_interfaces.h.md)
**Used by** — [`patrol_path.cpp`](patrol_path.cpp.md) · [`patrol_path.h`](patrol_path.h.md) · [`patrol_path_inline.h`](patrol_path_inline.h.md) · [`patrol_path_storage.cpp`](patrol_path_storage.cpp.md) · [`patrol_point.cpp`](patrol_point.cpp.md) · [`patrol_point_inline.h`](patrol_point_inline.h.md) · [`alife_monster_patrol_path_manager.cpp`](../../../xrGame/alife_monster_patrol_path_manager.cpp.md) · [`alife_smart_terrain_task.cpp`](../../../xrGame/alife_smart_terrain_task.cpp.md)
**Tier floor** — T2: a record with a load and save; the serialized form is the engine's own and versioned only against itself.

## Purpose

Declares the surface implemented in [`patrol_point.cpp`](patrol_point.cpp.md), with the accessors
in [`patrol_point_inline.h`](patrol_point_inline.h.md). A patrol point is one waypoint of an
authored route placed by a level designer: a position, a name by which scripts refer to it, a
flag word whose meaning is the game's, and — the part this module cares about — the two
navigation identities the position resolves to.

## State

```text
RECORD PatrolPoint
  name            : text          # authored; scripts address points by it
  position        : vector3       # world space, corrected onto the mesh at load
  flags           : int (32-bit)  # opaque here; the game layer interprets them
  level_vertex_id : int (32-bit)  # the mesh vertex the point sits on
  game_vertex_id  : int (16-bit)  # the game vertex covering that mesh vertex
```

**Invariants** — after loading, the point's position is on its mesh vertex and its game identity
is the one the cross table assigns to that mesh vertex; the three are consistent, which is what
"correcting" the position at load establishes. A point whose position is off the mesh keeps an
invalid mesh identity, and a debug build refuses it by name, because a patrol route with an
unreachable waypoint is an authoring error that must be caught in the editor rather than at
runtime.

## Exported units

- construction from authored data, taking the level mesh, the cross table and the game graph, so
  the identities can be resolved immediately.
- `load` / `save` — the serialized form: all five fields, identities included.
- `load_raw` — read the authored form (position, flags, name) and resolve the identities.
- `position()` / `flags()` / `name()` — field reads, valid only after the point is loaded.
- `level_vertex_id(mesh, cross, graph)` / `game_vertex_id(mesh, cross, graph)` — read an identity,
  with the graphs passed explicitly.
- `level_vertex_id()` / `game_vertex_id()` — the same, reaching the graphs through the global AI
  space; used by the game layer, which cannot pass them.

**Notes** — the two forms of each identity accessor are the file's one structural decision. The
explicit form is used from inside this module, where the graphs are at hand and where a point may
be being resolved against graphs other than the currently loaded ones — the level compiler does
exactly that. The implicit form exists for the game layer and additionally *re-resolves* the
identity when the point turns out to be on the currently loaded level, because a point loaded
from a save may carry identities from a different level's numbering.

Both serialized and authored forms exist because patrol routes appear twice: authored in the
level's data with positions only, and saved in a game snapshot with the resolved identities
already in them.
