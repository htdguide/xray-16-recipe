# src/xrAICore/Navigation/level_graph_vertex.cpp

> Walking the mesh along a straight line — can I get there, how far do I get, which cells does it cross — and the interpolation that turns four stored cover values into cover in an arbitrary direction.

**Needs** — [`level_graph.h`](level_graph.h.md) · [`level_graph_vertex_inline.h`](level_graph_vertex_inline.h.md) · [`game_level_cross_table.h`](game_level_cross_table.h.md) · [`../../xrCore/_fbox2.h`](../../xrCore/_fbox2.h.md)
**Used by** — [`level_graph.h`](level_graph.h.md)
**Tier floor** — T1: these are the mesh's hot queries; a creature runs several per frame.

## Purpose

Everything here answers a question of the form "if I walk from here in that direction, what
happens?". Creatures ask it constantly: to check line of movement before committing to a step, to
find how far they can retreat, to mark the cells a grenade's blast covers, to decide whether a
cover position is reachable in a straight line. None of it involves search — these are traversals
of the mesh along a line.

The file also holds the cover interpolation, which is unrelated to walking and is here because it
is a per-vertex computation like the rest.

## Stateless.

All operations read the mesh; the only writes are into caller-supplied output lists.

## The common walk

Four of the functions here are the same traversal with different bookkeeping, and recognizing that
is worth more to a rebuilder than four separate pseudocode blocks. The shape is:

```text
STEP walk(start_vertex, start_point, finish_point, per_step_action) -> distance_travelled
  current  <- start_vertex
  previous <- none
  travelled <- 0
  total <- plan distance from start_point to finish_point
  crossing <- start_point
  WHILE finish_point is not inside current's cell AND travelled < total + epsilon
    best <- none
    FOR EACH of current's four links
      next <- the linked vertex
      IF next is not valid OR next IS previous   CONTINUE
      choose_point(start_point, finish_point, current's contour, next, crossing, best)
    IF best IS none
      RETURN travelled                      # the line leaves the mesh here
    travelled <- plan distance from start_point to crossing
    previous  <- current
    current   <- best
    per_step_action(current)
  RETURN travelled
```

The four variants differ only in `per_step_action` and in what they return:

- **`check_position_in_direction`** — no action. On finishing, returns the plan distance to the
  destination if the final cell contains it *and* its surface height is within half a metre of the
  destination's height; otherwise returns a caller-supplied "too far" value. The height gate is
  what stops a creature from believing it can walk to a point directly above or below it.
- **`mark_nodes_in_direction`** — records each cell entered into a caller's list and optionally
  sets a flag in a caller's per-vertex bitmap. Returns how far the line got. Three entry points
  wrap it: by direction and length, by destination point, by destination vertex.
- **`farthest_vertex_in_direction`** — records the last cell reached, optionally marks, and
  optionally stops at the first inaccessible cell rather than walking through it. Returns how far
  it got. This is the "how far can I actually retreat" query.
- **`check_position_in_direction_slow`** and **`check_vertex_in_direction_slow`** — a different
  traversal; see below.

**Invariants** — every variant excludes the cell it just left from the next step's candidates,
and every variant stops when the accumulated distance exceeds the segment's length. Without both,
the walk can oscillate or run past the destination.

## `choose_point` — picking the next cell

**Contract** — given the current cell's contour, a candidate neighbour, and the segment being
walked, decide whether the segment crosses into that neighbour and if so where. Updates the
running crossing point and the chosen neighbour only when the candidate is better.

```text
FUNCTION choose_point(start, finish, current_contour, candidate_id, io_point, io_choice)
  candidate_contour <- contour of candidate_id
  shared <- the segment where the two contours overlap
  IF the walked segment does not cross `shared`   RETURN
  FOR EACH of the candidate contour's four edges
    kind, hit <- intersect(walked segment, that edge)
    IF kind IS proper_crossing
      IF hit is closer to `finish` than io_point is, within epsilon
        io_point <- hit ; io_choice <- candidate_id
    ELSE IF kind IS collinear_overlap
      # the segment runs along the edge: take the far end of the overlap
      pick whichever of the edge's two endpoints is further from `start`,
      provided it is further than io_point already is
      io_point <- it ; io_choice <- candidate_id
```

**Notes** — the two intersection outcomes are treated differently on purpose. A proper crossing is
scored by *closeness to the destination*, so among several candidates the walk takes the one that
advances furthest. A collinear overlap — the walked line lying exactly along a cell edge, which
happens whenever a creature walks along a grid axis — is scored by *distance from the start*,
because "closest to the destination" is degenerate when every point on the overlap is equally on
the line. Collapsing the two cases into one rule makes axis-aligned movement stall.

The candidate's *own* contour is tested edge by edge, but only after the shared segment between
the two contours has been confirmed crossed. The first test is the cheap rejection; the second
finds the actual point.

## The box-walk variants

**Contract** — `check_position_in_direction_slow` and `check_vertex_in_direction_slow` answer
"can I reach that position / that vertex by walking straight, without crossing anything masked
inaccessible", returning the destination vertex or false. `create_straight_path`'s point-emitting
form in this file answers the same traversal while emitting the crossing points.
`neighbour_in_direction` answers the one-step version: does any accessible neighbour lie in this
direction at all, probing four cell-widths ahead.

These use a different and cheaper traversal than `choose_point`: instead of contour geometry, each
candidate neighbour is represented by its axis-aligned plan square, grown from the cell centre by
half a cell, and the step is taken to the first neighbour whose square the ray pierces.

```text
STEP box_walk(start_vertex, start_2d, finish_2d)
  current <- start_vertex ; previous <- none
  best_sqr <- squared plan distance from current's centre to finish_2d
  LOOP
    FOR EACH of current's four links
      next <- linked vertex
      IF next is not valid OR next IS previous   CONTINUE
      square <- next's plan centre grown by half a cell
      IF the ray (start_2d, finish_2d - start_2d) does not pierce square   CONTINUE
      IF next's cell IS the destination cell     RETURN accessible(next) ? next : failure
      d <- squared plan distance from square's centre to finish_2d
      IF d > best_sqr                            CONTINUE      # monotonicity
      IF next is masked inaccessible             RETURN failure
      best_sqr <- d ; previous <- current ; current <- next
      BREAK
    IF no link was taken  RETURN failure
```

**Invariants** — the destination-cell test comes *before* the monotonicity test, so arriving is
never refused for moving away. Masked cells fail the whole query rather than being skipped: the
question is whether a straight walk is possible, and a fence across it means no.

**Notes** — the two traversals coexist because they answer subtly different questions. The contour
walk respects the cells' real tilted shapes and produces geometrically correct crossing points;
the box walk works purely in plan with squares and is faster but approximates the cells as
axis-aligned. Callers that need points use the first; callers that need only a yes-or-no use the
second. Their names call the second "slow", which is a leftover and the opposite of the truth.

## `distance(position, vertex)`

**Contract** — the squared distance from a world position to a vertex's cell, computed as the
minimum over the cell contour's four edges of the distance to that segment. Squared, not linear:
every caller compares distances and none needs the root.

## `cover_in_direction`

**Contract** — interpolate a vertex's four stored quadrant cover values into a cover value for an
arbitrary compass angle. Linear within each quadrant, between the two values bounding it.

```text
FUNCTION cover_in_direction(angle, b0, b1, b2, b3) -> real
  # rotate the angle into the first quadrant, rotating the four values with it
  SELECT quadrant of angle
    first  -> low, high <- b0, b1 ; local <- angle
    second -> low, high <- b1, b2 ; local <- angle - quarter_turn
    third  -> low, high <- b2, b3 ; local <- angle - half_turn
    fourth -> low, high <- b3, b0 ; local <- angle - three_quarter_turn
  RETURN (high - low) * local / quarter_turn + low
```

**Notes** — the callers pass the four cover values in a rotated order relative to how they are
stored, and the function's parameter names do not match its uses. Read it as: the four values are
the cover at four compass directions ninety degrees apart, and the result is the linear
interpolation between the two that bracket the requested direction. A rebuild should fix the
naming and keep the mapping, because the *assignment of stored quadrant to compass direction* is
fixed by the offline compiler that wrote them.
