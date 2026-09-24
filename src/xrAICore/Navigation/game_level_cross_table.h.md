# src/xrAICore/Navigation/game_level_cross_table.h

> Declares the table that maps every vertex of a level's fine navigation mesh to the coarse game vertex it belongs to — the join between the two graphs.

**Needs** — [`game_level_cross_table_inline.h`](game_level_cross_table_inline.h.md) · [`game_graph_space.h`](game_graph_space.h.md) · [`../../Common/LevelStructure.hpp`](../../Common/LevelStructure.hpp.md) · [Data: Level data](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`AISpaceBase.cpp`](../AISpaceBase.cpp.md) · [`patrol_point.cpp`](PatrolPath/patrol_point.cpp.md) · [`game_graph.h`](game_graph.h.md) · [`game_graph_inline.h`](game_graph_inline.h.md) · [`game_level_cross_table_inline.h`](game_level_cross_table_inline.h.md) · [`level_graph_vertex.cpp`](level_graph_vertex.cpp.md) · [`Level_network_spawn.cpp`](../../xrGame/Level_network_spawn.cpp.md) · [`ai_rat_behaviour.cpp`](../../xrGame/ai/monsters/rats/ai_rat_behaviour.cpp.md) · [`alife_dynamic_object.cpp`](../../xrGame/alife_dynamic_object.cpp.md) · [`alife_group_abstract.cpp`](../../xrGame/alife_group_abstract.cpp.md) · [`alife_monster_detail_path_manager.cpp`](../../xrGame/alife_monster_detail_path_manager.cpp.md) · [`alife_online_offline_group.cpp`](../../xrGame/alife_online_offline_group.cpp.md) · [`alife_switch_manager.cpp`](../../xrGame/alife_switch_manager.cpp.md) · [`level_changer.cpp`](../../xrGame/level_changer.cpp.md)
**Tier floor** — T1: a dense array read in place out of a mapped file; one entry per mesh vertex, so it is hundreds of thousands of entries and its layout is frozen.

## Purpose

Declares the surface implemented in
[`game_level_cross_table_inline.h`](game_level_cross_table_inline.h.md). The engine's two-level
navigation only works if a position can be translated between the levels: given a fine mesh
vertex, which coarse game vertex covers it? That question is asked constantly — every time an
entity moves, every time the alife simulation decides whether something is near something else —
so it is answered by a precomputed table rather than by a search.

The table is one entry per level-mesh vertex, so it is the same size as the mesh itself. It is
built offline and, from the *Clear Sky* generation onward, embedded in the game graph file;
older builds keep it as a separate file named `level.gct` in the level's directory.

## State

```text
RECORD CrossTableHeader
  version           : int (32-bit)     # checked against the engine's supported range
  level_vertex_count: int (32-bit)     # must equal the level mesh's vertex count
  game_vertex_count : int (32-bit)
  level_guid        : Guid             # must match the level mesh this table describes
  game_guid         : Guid             # must match the game graph it refers into

RECORD Cell                            # one per level-mesh vertex, in vertex order
  game_vertex_id : int (16-bit)        # the game vertex covering this mesh vertex
  distance       : real                # distance from the mesh vertex to that game vertex
```

**Invariants** — the cell array is indexed directly by level-mesh vertex identity, so the table
and the mesh must have been built together; the two identity stamps are what enforce that. Every
mesh vertex has exactly one covering game vertex — the mapping is total, not partial. The
distance is precomputed at build time and is the path distance, not the straight line.

## Exported units

- `CrossTable(buffer, size)` — attach to an already-mapped region, the embedded case.
- `CrossTable(file_name)` — open a standalone `level.gct`, the older case.
- `vertex(level_vertex_id)` — the cell for a mesh vertex. Bounds-checked against the header.
- `header()` — the loaded header.

**Notes** — the per-cell distance is what lets a creature answer "am I closer to game vertex A or
B" without searching, which is how the alife simulation decides which coarse vertex an entity
counts as being at. Two entities at different mesh vertices covered by the same game vertex are
in the same place as far as the off-screen simulation is concerned.
