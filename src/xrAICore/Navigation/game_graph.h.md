# src/xrAICore/Navigation/game_graph.h

> Declares the cross-level graph object — the loaded `game.graph` file plus the queries the alife simulation and the searches make of it.

**Needs** — [`game_graph_inline.h`](game_graph_inline.h.md) · [`game_graph_space.h`](game_graph_space.h.md) · [`game_level_cross_table.h`](game_level_cross_table.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`AISpaceBase.cpp`](../AISpaceBase.cpp.md) · [`path_manager_game.h`](PathManagers/path_manager_game.h.md) · [`path_manager_game_inline.h`](PathManagers/path_manager_game_inline.h.md) · [`path_manager_game_level.h`](PathManagers/path_manager_game_level.h.md) · [`path_manager_game_vertex.h`](PathManagers/path_manager_game_vertex.h.md) · [`patrol_point.cpp`](PatrolPath/patrol_point.cpp.md) · [`ai_object_location_impl.h`](ai_object_location_impl.h.md) · [`game_graph_inline.h`](game_graph_inline.h.md) · [`game_graph_script.cpp`](game_graph_script.cpp.md) · [`CustomMonster.cpp`](../../xrGame/CustomMonster.cpp.md) · [`GameObject.cpp`](../../xrGame/GameObject.cpp.md) · [`LevelGraphDebugRender.cpp`](../../xrGame/LevelGraphDebugRender.cpp.md) · [`base_monster_net.cpp`](../../xrGame/ai/monsters/basemonster/base_monster_net.cpp.md) · [`ai_rat.cpp`](../../xrGame/ai/monsters/rats/ai_rat.cpp.md) · _and 22 more_
**Tier floor** — T1: it holds a mapped file region and reads records in place out of it.

## Purpose

Declares the surface implemented in [`game_graph_inline.h`](game_graph_inline.h.md), with the
script export in [`game_graph_script.cpp`](game_graph_script.cpp.md). One instance exists per
running game, loaded from a file named `game.graph` in the game data.

It satisfies the same search-facing surface as every other graph in the engine — `begin`,
`value`, `edge_weight`, `is_accessible`, `valid_vertex_id` — so the same search walks it.

## State

Stateless. The loaded graph's state is in [`game_graph_inline.h`](game_graph_inline.h.md) and its on-disk records in [`game_graph_space.h`](game_graph_space.h.md).

## Exported units

- `GameGraph(file_name)` / `GameGraph(stream)` — load, taking ownership of the file or not.
- `header()` — the loaded header: version, counts, identity, and the table of levels.
- `vertex(id)` / `vertex_id(vertex)` / `valid_vertex_id(id)` / `set_invalid_vertex(out_id)`.
- `begin(id, first, last)` / `value(id, edge)` / `edge_weight(edge)` — the search-facing view.
- `begin_spawn(id, first, last)` — the death points attached to a vertex.
- `distance(from, to)` — the precomputed cost of the edge between two *adjacent* vertices; fails
  hard if they are not adjacent.
- `accessible(id)` / `accessible(id, value)` — per-vertex blocking, settable at runtime.
- `mask(requirement, actual)` — the four-byte terrain-type match.
- `set_current_level(level_id)` — bind the cross table for the level now loading, and find a
  game vertex belonging to it.
- `current_level_vertex()` — that vertex.
- `cross_table()` — the current level's level-vertex-to-game-vertex table.
- `save(stream)` — write the graph back out unchanged.

**Notes** — accessibility is the one mutable thing about the game graph: everything else is the
mapped file. It is marked as modifiable through a read-only handle, because closing a game vertex
off is something the game logic does to a graph it otherwise treats as constant. In a rebuild the
honest shape is an immutable graph plus a separate per-vertex blocking bitmap owned by whoever
blocks.
