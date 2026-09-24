# src/xrAICore/Navigation/level_graph_space.h

> Names the level mesh's header, its vertex, and the two geometric shapes the mesh's queries are phrased in.

**Needs** — [`../../Common/LevelStructure.hpp`](../../Common/LevelStructure.hpp.md) · [Data: Level data](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`ai_object_location.h`](ai_object_location.h.md) · [`level_graph.h`](level_graph.h.md) · [`level_graph_inline.h`](level_graph_inline.h.md) · [`level_graph_manager.h`](level_graph_manager.h.md) · [`level_graph_vertex_inline.h`](level_graph_vertex_inline.h.md)
**Tier floor** — T1: the header and vertex are the on-disk records of `level.ai`, read in place.

## Purpose

A thin naming layer: it gives the raw on-disk records of the level navigation mesh the names the
rest of the engine uses, and adds the two geometric shapes the mesh's spatial queries traffic in.
The records themselves — their bit packing, their generations — belong to
[`../../Common/LevelStructure.hpp`](../../Common/LevelStructure.hpp.md); the behaviour belongs to
[`level_graph.h`](level_graph.h.md). This file exists only so those two need not know each other's
spelling, and a rebuild may fold it into either.

## State

```text
RECORD LevelMeshHeader          # the first bytes of the level.ai file
  version    : int (32-bit)     # the mesh generation; see the version ladder
  count      : int (32-bit)     # number of vertices
  cell_size  : real             # the mesh's square cell edge, in metres
  factor_y   : real             # the height range the packed vertical coordinate spans
  box        : box3             # the level's bounding box; the origin for all packed positions
  guid       : Guid             # must match the cross table and game graph built alongside

RECORD LevelVertex              # fixed size; the vertex array follows the header
  links      : list<int>        # exactly 4, packed into a shared bit field: the neighbours
                                # north/east/south/west; an out-of-range value means "no
                                # neighbour in that direction"
  high_cover : list<int>        # exactly 4 nibbles, one per quadrant
  low_cover  : list<int>        # exactly 4 nibbles, one per quadrant
  plane      : int (16-bit)     # compressed surface normal
  position   : PackedPosition   # packed cell index and height; see below

RECORD PackedPosition
  xz : int                      # cell index = x * row_length + z, one packed field
  y  : int (16-bit)             # height as a fraction of factor_y above the box floor

RECORD Segment                  # two points
  v1, v2 : vector3

RECORD Contour EXTENDS Segment  # four points: a mesh cell's corners, projected onto its plane
  v3, v4 : vector3
```

**Invariants** — the vertex array is sorted by packed cell index, which is what makes "which
vertex is at this x,z" a binary search. Several vertices may share one cell index — they are
stacked floors at the same horizontal position — and they are then adjacent in the array, which
is what makes "which of the stacked vertices is at this height" a short linear scan from the
first match.

A vertex's four links are directional and their *order is load-bearing*: the straight-line walk
in [`level_graph_inline.h`](level_graph_inline.h.md) picks which edge of the cell a path crosses
purely from which link index it followed.

## Exported units

- `Header` — version, vertex count, cell size, height factor, bounding box, identity.
- `LevelVertex` — link, cover, plane and position accessors, plus ordering by packed cell index.
- `Position` — the packed position record.
- `Segment` / `Contour` — the shapes the mesh's geometry is phrased in.

**Notes** — a vertex's *cover* is two sets of four values, high and low, one per compass quadrant,
each a nibble that the engine reads as a fraction of fifteen. They describe how much of the view
from that vertex is blocked, at standing height and at crouching height, and they are precomputed
by the offline level compiler. The engine never computes them, only interpolates between them
(see [`level_graph_vertex_inline.h`](level_graph_vertex_inline.h.md)). That four-quadrant
resolution is the entire spatial fidelity the cover system has, and every "take cover" decision a
creature makes is built on it.
