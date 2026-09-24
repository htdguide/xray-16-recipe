# src/xrAICore/Navigation/level_graph_inline.h

> The level mesh's coordinate system — packing world positions into cell indices and back — plus cell containment, the accessibility fence, and the straight-line walk that turns a searched path into something a creature can follow.

**Needs** — [`level_graph.h`](level_graph.h.md) · [`level_graph_space.h`](level_graph_space.h.md) · [`level_graph_vertex_inline.h`](level_graph_vertex_inline.h.md) · [`../../xrCore/_fbox2.h`](../../xrCore/_fbox2.h.md)
**Used by** — [`level_graph.h`](level_graph.h.md)
**Tier floor** — T1: every accessor here is on the movement hot path and reads packed fields out of a mapped record.

## Purpose

Three things live here, and only the third is an algorithm.

The **coordinate system**: the mesh stores positions packed into a cell index and a quantized
height, and essentially every query converts between that and world space. Getting the conversion
exactly right — including its rounding — is a hard requirement, because the packing must
round-trip against data the offline compiler wrote.

The **accessibility fence**: a per-vertex flag that restrictors raise to keep a creature inside or
outside a volume.

The **straight-line walk**: given a start vertex and a straight segment in plan, produce the
sequence of cells the segment crosses and the point at which it crosses each boundary. This is
what a creature actually walks: a searched path gives cell-to-cell waypoints, and this turns a
leg of it into a polyline.

## State

Stateless. Everything here reads the mesh state described in [`level_graph.cpp`](level_graph.cpp.md); the only writes are the accessibility mask and caller-supplied output lists.

## The packing

**Contract** — a world position packs to a cell index and a quantized height, both relative to the
mesh's bounding box. Unpacking reverses it. The rounding is *round-to-nearest*, expressed as a
floor of the value plus a half, and a rebuild must reproduce that exactly.

```text
FUNCTION pack(position) -> PackedPosition
  ix <- floor((position.x - box.min.x) / cell_size + 0.5)
  iz <- floor((position.z - box.min.z) / cell_size + 0.5)
  xz <- ix * row_length + iz                     # one field, not two
  y  <- floor(65535 * (position.y - box.min.y) / factor_y + small_epsilon)
  y  <- clamp(y, 0, 65535)
  RETURN { xz, y }

FUNCTION unpack_xz(packed) -> (x, z)             # grid indices
  x <- packed.xz / row_length
  z <- packed.xz MOD row_length

FUNCTION unpack_position(packed) -> vector3      # world space, cell centre
  (ix, iz) <- unpack_xz(packed)
  x <- ix * cell_size + box.min.x
  z <- iz * cell_size + box.min.z
  y <- (packed.y / 65535) * factor_y + box.min.y
  RETURN { x, y, z }
```

**Invariants** — the height is quantized to sixteen bits across the box's full vertical extent,
so its resolution is the level's height divided by 65536 — a few tenths of a millimetre for a
typical level, which is why nothing worries about it. The horizontal packing is *exact* in the
sense that unpacking gives the cell's centre, not the original position: horizontal precision is
one cell, and every query that needs better works from the vertex's contour and plane instead.

The cell index has a maximum imposed by the record's packing width, and a position packing above
it is rejected rather than wrapped. That check is what `valid_vertex_position` exists for.

## `valid_vertex_position`

**Contract** — whether a world position can be packed at all: inside the bounding box with a
half-cell margin on each horizontal side, within the derived grid dimensions, and below the
packing width's ceiling. Every lookup that could otherwise index out of bounds asks this first.

## `inside`

**Contract** — whether a position lies in a vertex's cell, in several argument shapes. The plain
form compares packed cell indices only — it is a *horizontal* test and ignores height entirely.
The tolerance form additionally requires the vertex's stored height and the position's height to
agree within a caller-given margin.

**Notes** — the plain form being horizontal-only is the most easily misread thing in this file. A
position thirty metres above a vertex is "inside" it. Every caller that cares about storeys must
use the tolerance form or check the plane height itself, and
[`level_graph.cpp`](level_graph.cpp.md) does exactly that.

## `vertex_plane_y`

**Contract** — the height of a vertex's surface directly above or below a horizontal position.
Decompress the vertex's stored normal, build the plane through the vertex's unpacked centre, and
intersect a vertical ray with it.

**Notes** — this, not the stored height, is what every height comparison in the mesh uses. The
stored height is the cell centre's; cells are tilted, so a query at the cell's edge can differ
from it by half a cell's slope, which on a ramp is enough to pick the wrong floor.

## Adjacency

**Contract** — a vertex's neighbour range is always the four link slots, and `value` reads the
link at an index. Presented as an iterable range rather than four fields so that the same search
walks this mesh and the graphs that have variable degree. A link whose value does not validate
means "no neighbour in that direction".

## The accessibility mask

**Contract** — `is_accessible` is "the identity is valid and the vertex is not masked". Setting
and clearing come in checked and unchecked forms, singly and over a list of identities; the
checked forms assert the flag is not already in the state being set, which catches a restrictor
that masks the same vertex twice and would then unmask it once too few.

**Notes** — the naming is inverted relative to the meaning: *setting* the mask makes a vertex
inaccessible and *clearing* it makes it accessible. A rebuild should name these for what they do.

## `create_straight_path` — the walk

**Contract** — from a start vertex, follow a straight segment in plan and emit the path it traces
across the mesh: at every cell boundary crossed, the crossing point and the cell entered. Returns
whether the destination was reached. Fails — returning false and leaving a partial path — when
the segment leaves the mesh, when it reaches a cell that is masked inaccessible, or when no
neighbour continues the line. Optionally clears the output first, optionally emits the start
point, and optionally computes each point's height from the cell's plane.

```text
FUNCTION create_straight_path(start_vertex, start_2d, finish_2d, out, assign_y) -> bool
  IF finish_2d is not a valid mesh position       RETURN false
  destination_cell <- pack_xz(finish_2d)
  current <- start_vertex ; previous <- start_vertex
  direction <- finish_2d - start_2d
  best_sqr <- squared plan distance from current's centre to the destination

  IF start_vertex's cell IS destination_cell      # both ends in one cell: one point
    emit(finish_2d, start_vertex) ; RETURN true

  LOOP
    advanced <- false
    FOR link_index IN 0 .. 3
      next <- current.link(link_index)
      IF next IS previous OR next is not valid    CONTINUE
      cell_box <- the square of side cell_size centred on next's plan centre
      IF NOT cell_box is crossed by the ray (start_2d, direction)   CONTINUE

      d <- squared plan distance from cell_box's centre to finish_2d
      IF d > best_sqr AND next's cell IS NOT destination_cell
        CONTINUE                                  # moving away and not the goal: reject
      IF next is masked inaccessible              RETURN false
      best_sqr <- d

      edge <- the two corners of cell_box selected by link_index   # see Notes
      crossing <- intersection of the segment (start_2d, finish_2d) with that edge,
                  clamped into the edge's own extent
      IF assign_y  crossing.y <- plane_y(current, crossing.x, crossing.z)
      emit(crossing, next)

      IF next's cell IS destination_cell
        p <- finish_2d
        IF assign_y  p.y <- plane_y(current, p.x, p.z)
        emit(p, next) ; RETURN true

      advanced <- true ; previous <- current ; current <- next
      BREAK
    IF NOT advanced  RETURN false
```

**Invariants** — the walk is monotone: a step that increases the plan distance to the destination
is refused unless it lands in the destination cell itself. That single rule is what terminates the
loop, since the mesh offers no other guarantee that following crossed boundaries makes progress.
The previous vertex is excluded from each step's candidates, which prevents the immediate
two-cell oscillation the monotonicity rule alone would still allow.

The link index selects which pair of the cell's plan corners the crossing is computed against —
link zero is the far edge in one direction, link one the near edge, and so on. This is where the
*order* of a vertex's four links becomes load-bearing: with the links permuted, every crossing
point lands on the wrong edge and paths visibly cut corners through walls.

**Notes** — heights are computed from the plane of the cell being *left*, not the cell being
entered. At a boundary between two differently tilted cells the two differ; choosing the leaving
cell means the emitted polyline stays on the surface the creature is currently walking, and only
steps to the new plane at the next point. A rebuild that uses the entered cell produces a path
that dips or rises just before each boundary.

The crossing point is clamped into the edge's extent after intersection. The unclamped
intersection can fall slightly outside it through floating-point error on a nearly parallel
segment, and an unclamped point outside the cell makes the next iteration's monotonicity check
nonsense.

A debug build aborts if the emitted path exceeds a hundred thousand points, on the grounds that
the walk has failed to terminate. That is a safety net over the monotonicity argument, not a
supported path length.

## `assign_y_values`

**Contract** — given a path of points each tagged with the vertex it belongs to, set every point's
height from that vertex's surface plane. Caches the plane across consecutive points sharing a
vertex, which is the common case. Used to lift a path computed purely in plan onto the mesh's
surface.

## `iterate_vertices`

**Contract** — call a predicate for every vertex whose packed cell index falls between the cells
of two world positions. Because the array is sorted by cell index, the range is a contiguous slice
found by two binary searches; a bound that is not a valid mesh position degenerates to the array's
own end.

**Notes** — this is a *cell-index* range, not a rectangle. Since the index is `x * row_length + z`,
the slice covers every cell between the two in row-major order, which for a box query includes
everything in the intervening rows. Callers filter. A rebuild should not mistake this for a
spatial query.

The upper bound is advanced by one past the found position when it is not already at the end,
which makes the range inclusive of the last matching cell. Removing that adjustment silently drops
the final row.
