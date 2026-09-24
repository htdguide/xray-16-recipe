# src/xrAICore/Navigation/level_graph.h

> Declares the level navigation mesh — a sorted grid of walkable cells with four-way links, and the whole vocabulary of spatial questions the AI asks of it.

**Needs** — [`level_graph.cpp`](level_graph.cpp.md) · [`level_graph_inline.h`](level_graph_inline.h.md) · [`level_graph_vertex.cpp`](level_graph_vertex.cpp.md) · [`level_graph_vertex_inline.h`](level_graph_vertex_inline.h.md) · [`level_graph_space.h`](level_graph_space.h.md) · [`level_graph_manager.h`](level_graph_manager.h.md) · [`game_graph_space.h`](game_graph_space.h.md) · [`../../Common/LevelStructure.hpp`](../../Common/LevelStructure.hpp.md) · [`../../xrCore/_plane.h`](../../xrCore/_plane.h.md)
**Used by** — [`AISpaceBase.cpp`](../AISpaceBase.cpp.md) · [`path_manager_level.h`](PathManagers/path_manager_level.h.md) · [`path_manager_level_inline.h`](PathManagers/path_manager_level_inline.h.md) · [`path_manager_level_straight_line_inline.h`](PathManagers/path_manager_level_straight_line_inline.h.md) · [`patrol_point.cpp`](PatrolPath/patrol_point.cpp.md) · [`ai_object_location_impl.h`](ai_object_location_impl.h.md) · [`ai_object_location_inline.h`](ai_object_location_inline.h.md) · [`level_graph.cpp`](level_graph.cpp.md) · [`level_graph_inline.h`](level_graph_inline.h.md) · [`level_graph_vertex.cpp`](level_graph_vertex.cpp.md) · [`level_graph_vertex_inline.h`](level_graph_vertex_inline.h.md) · [`CustomMonster.cpp`](../../xrGame/CustomMonster.cpp.md) · [`GameObject.cpp`](../../xrGame/GameObject.cpp.md) · [`LevelGraphDebugRender.cpp`](../../xrGame/LevelGraphDebugRender.cpp.md) · _and 79 more_
**Tier floor** — T1: the mesh is a mapped file region read in place, hundreds of thousands of packed records.

## Purpose

Declares the surface implemented across four files. The level mesh is the fine half of the
engine's two-level navigation: one per loaded level, a grid of square walkable cells each knowing
its four neighbours, its surface plane, and how much cover it offers in each quadrant. Everything
a creature does in the detailed simulation is expressed in its vertices.

The substance is split by kind rather than by size:

- [`level_graph.cpp`](level_graph.cpp.md) — loading, and finding the vertex under a position.
- [`level_graph_inline.h`](level_graph_inline.h.md) — the coordinate system: packing and
  unpacking positions, cell containment, the accessibility mask, and the straight-line walk.
- [`level_graph_vertex.cpp`](level_graph_vertex.cpp.md) — walking the mesh in a direction:
  raycasts against the mesh, marking, the point-list form of a straight path, cover in a
  direction.
- [`level_graph_vertex_inline.h`](level_graph_vertex_inline.h.md) — the geometry primitives those
  walks are built from: segment intersection, cell contours, nearest point on a contour, and the
  cover integrals.

## State

Stateless. The mesh's state is in [`level_graph.cpp`](level_graph.cpp.md) and its on-disk records in [`level_graph_space.h`](level_graph_space.h.md).

## The vocabulary

A rebuilder should learn these words before reading the four twins, because they recur
everywhere.

- **vertex** — one walkable mesh cell. Identified by its index in a sorted array.
- **packed position** — a cell index (`x * row_length + z`) and a quantized height. The mesh
  stores positions this way and the engine converts constantly.
- **contour** — a vertex's four corners, projected onto its surface plane. The mesh is a grid in
  plan but the cells are tilted, so a "cell" is a quadrilateral in space.
- **plane y** — the height of a vertex's surface directly above or below a given horizontal
  position. This is how the mesh answers "how high is the floor here".
- **cover** — the four-quadrant occlusion values, high and low, precomputed per vertex.
- **accessibility mask** — a per-vertex runtime flag; restrictors set it to fence creatures in or
  out. Distinct from a vertex not existing.
- **straight path** — the sequence of cells a straight line crosses, and the crossing points.
  This is what a creature actually walks along between waypoints of a searched path.

## Exported units

Grouped by what they answer; each group's contract is in the twin named above.

**Identity and bounds** — `valid_vertex_id`, `vertex(id)`, `vertex(record)`, `vertex_id(record)`,
`set_invalid_vertex`, `begin`/`end` over the vertex array, `header`, `max_x`, `max_z`,
`row_length`, `level_id` (read and write).

**Position** — `vertex_position` in all its directions (packed to world, world to packed, by
vertex, by identity), `unpack_xz` to grid indices or to world coordinates, `vertex_plane_y`,
`valid_vertex_position`, `v2d`/`v3d` between the plan and space.

**Containment** — `inside(vertex, position)` in packed, world and plan forms, with and without a
height tolerance.

**Lookup** — `vertex_id(position)` (the binary search by cell, then the best stacked vertex by
height), `vertex(position)` (exhaustive nearest, the slow fallback), `vertex(current, position)`
(the incremental one every caller actually uses).

**Adjacency** — `begin(vertex, first, last)` and `value(vertex, index)`: the four links, exposed
in the same shape every other graph uses so the same search walks them.

**Accessibility** — `is_accessible`, and `set_mask`/`clear_mask` in checked and unchecked forms,
singly and over a list.

**Distance and geometry** — `distance` between vertices, between a position and a vertex, and
from a point to a segment; `contour`, `nearest`, `intersect`, `similar`, `project_point`.

**Walking** — `check_vertex_in_direction`, `check_position_in_direction`,
`mark_nodes_in_direction`, `farthest_vertex_in_direction`, `create_straight_path` in three forms,
`assign_y_values`, `neighbour_in_direction`, `choose_point`, `iterate_vertices`.

**Cover** — `vertex_high_cover`, `vertex_low_cover`, `high_cover_in_direction`,
`low_cover_in_direction`, `compute_high_square`, `compute_low_square`, `square`,
`cover_in_direction`, `vertex_high_cover_angle`, `vertex_low_cover_angle`.

**Notes** — the class is enormous and its size is itself informative: the level mesh is not a
data structure with a few queries, it is the engine's entire spatial reasoning surface. A rebuild
should keep the grouping above and may well split it into four types — an array of cells, a
coordinate system, a walker, and a cover model — none of which the present code separates.

A `vertex` name is overloaded four ways here — the vertex record type, the accessor by identity,
the reverse accessor by record, and the exhaustive nearest-vertex search. A rebuild should give
the last of those a different name; the source's own call sites are hard to read because of it.
