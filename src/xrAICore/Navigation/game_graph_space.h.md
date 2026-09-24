# src/xrAICore/Navigation/game_graph_space.h

> The frozen byte layout of the cross-level navigation graph — the coarse graph that spans every level in the game and is what the off-screen simulation moves entities along.

**Needs** — [`../../xrCore/Containers/AssociativeVector.hpp`](../../xrCore/Containers/AssociativeVector.hpp.md) · [`../../xrCore/FixedVector.h`](../../xrCore/FixedVector.h.md) · [`../../Common/GUID.hpp`](../../Common/GUID.hpp.md) · [Data: Level data](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`path_manager_game_vertex_inline.h`](PathManagers/path_manager_game_vertex_inline.h.md) · [`path_manager_params_game_vertex.h`](PathManagers/path_manager_params_game_vertex.h.md) · [`patrol_path_params.h`](PatrolPath/patrol_path_params.h.md) · [`patrol_point.h`](PatrolPath/patrol_point.h.md) · [`ai_object_location.h`](ai_object_location.h.md) · [`game_graph.h`](game_graph.h.md) · [`game_graph_inline.h`](game_graph_inline.h.md) · [`game_level_cross_table.h`](game_level_cross_table.h.md) · [`level_graph.h`](level_graph.h.md) · [`alife_monster_detail_path_manager.h`](../../xrGame/alife_monster_detail_path_manager.h.md) · [`alife_monster_patrol_path_manager.h`](../../xrGame/alife_monster_patrol_path_manager.h.md) · [`alife_online_offline_group_brain.h`](../../xrGame/alife_online_offline_group_brain.h.md) · [`alife_simulator_base.h`](../../xrGame/alife_simulator_base.h.md) · [`alife_smart_terrain_task.h`](../../xrGame/alife_smart_terrain_task.h.md) · _and 11 more_
**Tier floor** — T1: these records are read by pointing at a mapped file region, so every field's offset and width is frozen by the shipped game data.

## Purpose

The engine has two navigation graphs. The *level graph* is a fine mesh covering one loadable
level, described in [`level_graph.h`](level_graph.h.md). The *game graph* is a single coarse
graph spanning every level in the game at once — a few thousand vertices for the whole world
against a few hundred thousand for one level. A creature standing anywhere is simultaneously at
one level vertex and at one game vertex; the alife simulation moves off-screen entities along
game-graph edges without ever touching a level mesh, and a creature crossing between levels does
so along a game edge whose two endpoints carry different level identifiers.

This file declares the records as they sit in the file, and nothing else. Their behaviour is in
[`game_graph_inline.h`](game_graph_inline.h.md).

## State

```text
RECORD GameGraphHeader          # written first, variable length because of the level table
  version          : int (8-bit)
  vertex_count     : int (16-bit)      # hence at most 65535 game vertices for the whole game
  edge_count       : int (32-bit)
  death_point_count: int (32-bit)
  guid             : Guid              # must match the level files built alongside it
  level_count      : int (8-bit)       # then that many Level records
  levels           : map<LevelId, Level>

RECORD Level                    # variable length: two length-prefixed strings
  name    : text                       # the level's folder name, used to locate its data
  offset  : vector3                    # where this level sits in world space
  id      : int (8-bit)                # hence at most 255 levels
  section : text                       # the configuration section describing the level
  guid    : Guid

RECORD GameVertex               # fixed size; the vertex array follows the header
  local_point      : vector3           # position within its own level
  global_point     : vector3           # position in world space (local plus the level offset)
  level_id         : int (8 bits of a packed 32-bit word)
  level_vertex_id  : int (24 bits of the same word)   # the level-mesh vertex under this one
  vertex_types     : list<int (8-bit)> # exactly 4; the terrain-type mask, see below
  edge_offset      : int (32-bit)      # byte offset from the start of the vertex array
  point_offset     : int (32-bit)      # byte offset to this vertex's death points
  neighbour_count  : int (8-bit)       # hence at most 255 edges out of one game vertex
  death_point_count: int (8-bit)

RECORD Edge                     # packed; the edge array follows the vertex array
  vertex_id : int (16-bit)             # destination game vertex
  distance  : real                     # precomputed path length between the two vertices

RECORD LevelPoint               # a "death point": an authored spawn position near a game vertex
  point           : vector3
  level_vertex_id : int (32-bit)
  distance        : real

RECORD TerrainPlace
  mask : list<LocationId>              # exactly 4 entries
```

**Invariants** — `level_vertex_id` is 24 bits, which caps a level's navigation mesh at about
16.7 million vertices *as referenced from the game graph*, even though the mesh itself numbers
vertices in 32 bits. The two are packed into one 32-bit word with the level identifier, which is
why the cap exists at all.

Edge and death-point offsets are byte offsets *from the start of the vertex array*, not from the
start of the file and not indices. That is what makes the whole graph usable by pointing at the
mapped region: follow an offset and you are at the data. A rebuild that parses the file into its
own structures must convert them to indices at load, once.

The `vertex_count` being 16-bit is why an invalid game vertex is represented as all-ones in 16
bits, and why that value must never be a real vertex.

## The terrain-type mask

Each game vertex carries four bytes describing the kind of place it is; each authored consumer
carries four bytes of requirement. A vertex satisfies a requirement when, for each of the four
positions, either the values match or the requirement byte is 255, which reads as "don't care".
Four independent categories is an authoring decision made in the level tools, and the engine
attaches no meaning to any of them — it only matches.

**Notes** — *death points* are authored positions attached to a game vertex, used by the alife
simulation to place things near a vertex rather than exactly on it. They are stored as a flat
array with each vertex holding an offset and a count into it, the same arrangement as the edges.

Two record shapes here are declared as writable structures in the offline level compiler and as
read-only in the game, from the same file. That is a build-time distinction only: the layout is
identical either way, and a rebuild should make the records plain data and put the writability
in the tool.
