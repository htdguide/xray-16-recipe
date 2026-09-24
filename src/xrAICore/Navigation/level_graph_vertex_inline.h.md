# src/xrAICore/Navigation/level_graph_vertex_inline.h

> The mesh's geometry primitives — segment intersection with an equality case, a cell's projected contour, nearest point on a contour — and the cover model that integrates a vertex's four stored occlusion values over a field of view.

**Needs** — [`level_graph.h`](level_graph.h.md) · [`level_graph_space.h`](level_graph_space.h.md) · [`../../xrCore/_plane.h`](../../xrCore/_plane.h.md)
**Used by** — [`level_graph.h`](level_graph.h.md) · [`level_graph_inline.h`](level_graph_inline.h.md) · [`level_graph_vertex.cpp`](level_graph_vertex.cpp.md)
**Tier floor** — T1: these run per cell crossed, several times per frame per creature.

## Purpose

Two unrelated bodies of work sharing a file.

The **geometry** the mesh walks are built from: where two segments cross, what a cell's four
corners are once projected onto its tilted surface, and the nearest point on that contour to a
given position. These are small, but their edge cases are exactly what the walks depend on.

The **cover model**: a vertex stores four occlusion values per height, at four compass
directions. Everything the AI knows about whether a position is exposed comes from interpolating
and integrating those four numbers, and this file is the entire model.

## Stateless.

## `intersect` — segment crossing with three outcomes

**Contract** — intersect two plan segments, reporting one of: no intersection, a proper crossing
(with the point), or *equality* — the two segments lie on the same line. Distinguishing the third
case is the point of the function: the mesh walks run along cell edges constantly, and a plain
crossing test either misses those or returns garbage.

```text
FUNCTION intersect(seg_a, seg_b) -> (outcome, point)
  build the line through seg_a as a, b, c with a*x + b*y + c = 0
  r3, r4 <- signed evaluations of seg_b's endpoints against that line
  IF r3 and r4 have the same non-zero sign, both clear of zero
    RETURN none
  build the line through seg_b likewise
  r1, r2 <- signed evaluations of seg_a's endpoints against it
  IF r1 and r2 have the same non-zero sign, both clear of zero
    RETURN none
  IF r1*r2 is zero AND r3*r4 is zero
    RETURN equal                     # every endpoint lies on the other line
  denom <- the two lines' cross term
  IF denom is zero
    RETURN collinear                 # parallel; no single point
  RETURN crossing, the point solved from the two line equations
```

**Invariants** — the same-side rejections are made tolerant: a product that is positive but whose
factors are near zero is *not* rejected, because an endpoint lying on the line is exactly the
case being looked for. A rebuild that tests `r3 * r4 > 0` without the near-zero exemptions turns
every along-an-edge walk into a dead end.

**Notes** — "no intersection" and "collinear" are deliberately given the same numeric value in the
source's outcome set, so a caller testing "did it intersect" treats collinear as a miss while a
caller switching on the outcome can still tell equality apart. A rebuild should make the three
outcomes distinct and have callers say which they mean.

## `intersect_no_check`

**Contract** — the same computation with the two same-side rejections removed: the caller has
already established that the segments cross, so the only questions are where, and whether the
case is degenerate. Both degenerate cases — endpoints on the line, and parallel lines — return the
second segment's far endpoint as the answer rather than failing.

**Notes** — used by the straight-line walk, which knows the segment enters the cell because the
cell's plan box was already pierced. Returning the far endpoint on a degenerate case keeps the
walk moving instead of stalling on a numerically awkward step; it is an approximation, and it is
bounded by half a cell.

## `contour`

**Contract** — a vertex's four corners in world space: the plan square of side `cell_size`
centred on the vertex, with each corner raised or lowered onto the vertex's surface plane. Corner
order is fixed — minimum-x/minimum-z, maximum-x/minimum-z, maximum-x/maximum-z, minimum-x/
maximum-z — and every consumer of a contour depends on that order.

```text
FUNCTION contour(vertex) -> Contour
  centre <- unpack_position(vertex.position)
  plane  <- through centre with the vertex's decompressed normal
  half   <- cell_size / 2
  four corners at (±half, ±half) in plan around centre, each at centre's height
  project each straight down or up onto plane
```

**Notes** — projecting onto the plane rather than sampling a height per corner is what makes a
cell a flat quadrilateral. Two adjacent cells with different tilts therefore do not share an
edge exactly — their contours meet along a line but at slightly different heights. Every walk in
the mesh tolerates that by working in plan and only consulting heights through
`vertex_plane_y`.

## `project_point`

**Contract** — move a point vertically until it lies on a plane. Undefined for a plane with no
vertical component; mesh surface planes always have one, because a vertical cell is not walkable.

## `nearest`

**Contract** — the nearest point on a segment to a position, and the squared distance to it,
clamped to the segment's endpoints. The contour form takes the minimum over the contour's four
edges. Squared throughout — nothing here needs a root.

## `inside(point, contour)` and `similar`

**Contract** — `inside` is an axis-aligned plan test against the contour's first and third
corners, with an epsilon margin: it relies on the fixed corner order and ignores the contour's
tilt. `similar` is plan equality within an epsilon, used to tell whether two corners of adjacent
cells are the same corner.

## `intersect(segment, contour_edge)` — the corner-aware form

**Contract** — intersect a segment with a cell edge in three dimensions, snapping to a corner
when an endpoint is within five centimetres of one, and otherwise interpolating the crossing
point's height along the edge by its plan position.

**Notes** — the five-centimetre corner snap exists because a path that passes almost exactly
through a cell corner otherwise produces a crossing point whose barycentric interpolation is
numerically unstable. The interpolation itself averages the two axes' parameters, falling back to
whichever axis is non-degenerate when the edge is axis-aligned. The constant is arbitrary in the
source, but it is comfortably below the cell size and comfortably above the mesh's positional
precision.

## The cover model

A vertex stores four occlusion values per height — standing and crouching — each a nibble read
as a fraction of fifteen. They are the *openness* at four compass directions, precomputed
offline. Three derived quantities are built on them.

### `square` — the integral of a linearly interpolated openness

**Contract** — the area swept by an openness value that varies linearly from one endpoint value
to another, over an angular extent. This is the model's kernel: the "amount of exposure" over an
arc is the integral of the openness over that arc, and the openness between two stored directions
is taken to be linear.

```text
FUNCTION square(near_value, far_value, extent = quarter_turn) -> real
  slope     <- 2 * (far_value - near_value) / half_turn
  intercept <- near_value
  RETURN   extent^3 * slope^2 / 6
         + extent^2 * slope * intercept / 2
         + extent   * intercept^2 / 2
```

**Notes** — the integrand is the *square* of the interpolated openness, not the openness itself —
which is what the cubic and the squared slope say. Reading it as an area swept by a radius that
varies linearly with angle makes the formula obvious: the area of a sector of radius `r` over an
angle `dθ` is `r² dθ / 2`, and integrating that with `r` linear in `θ` gives exactly these three
terms. So "cover" here is measured in units of *area visible*, not of distance or fraction.

### `compute_square` — exposure over a field of view

**Contract** — the total exposure of a vertex looking in a given direction with a given half-angle
of view. Rotates the four stored values so the direction falls in the first quadrant, then
integrates across however many quadrant boundaries the field of view spans, adding and — in one
case — subtracting sector areas so the arc is covered exactly once.

**Invariants** — the field of view may span more than one quadrant, and the four cases in the
source enumerate the ways it can straddle the boundaries. The one subtraction handles a field of
view lying entirely within the first quadrant but not containing its lower edge, where the
straightforward sum would double-count.

**Notes** — every call in the engine passes a half-angle of a quarter turn, i.e. a 180-degree
field of view. The general form is available and unused.

### `vertex_high_cover` / `vertex_low_cover`

**Contract** — the total exposure of a vertex in all directions: the sum of the four quadrant
integrals. One number per vertex per height, and the usual "how exposed is this position"
measure.

### `vertex_high_cover_angle` / `vertex_low_cover_angle`

**Contract** — sweep the full circle in caller-given angular steps and return the direction whose
half-circle exposure best satisfies a caller-supplied comparison. With "less than" it finds the
direction of best cover; with "greater than", the direction of clearest view.

```text
FUNCTION cover_angle(vertex_id, step, better) -> real
  best_angle <- 0
  best_value <- exposure(vertex_id, facing 0, half-angle quarter_turn)
  FOR angle FROM step TO full_turn STEP step
    value <- exposure(vertex_id, facing angle, half-angle quarter_turn)
    IF better(value, best_value)
      best_value, best_angle <- value, angle
  RETURN best_angle
```

**Notes** — ties keep the *earlier* angle, since the comparison is strict. With a coarse step and
a symmetric vertex that biases the answer towards zero — due east in the engine's convention.
Nothing depends on the bias, but a rebuild that reverses the comparison to non-strict will pick
different cover directions on flat ground and the difference is visible in play.

## `check_position_in_direction` / `check_vertex_in_direction` — the fast paths

**Contract** — before running a walk, answer the trivial cases directly: a destination position
already inside the start cell, or a destination vertex equal to the start vertex, succeeds with no
traversal at all. Otherwise delegate to the box walk in
[`level_graph_vertex.cpp`](level_graph_vertex.cpp.md). Both take world or plan coordinates.
