# src/xrGame/ai_obstacle.cpp

> Fits a tight oriented box around one dynamic object's visible bones and converts it into the set of navigation vertices that object blocks, so the pathfinder can route around a body that was never part of the level's static geometry.

**Needs** — [`ai_obstacle.h`](ai_obstacle.h.md) · [`ai_obstacle_inline.h`](ai_obstacle_inline.h.md) · [`ai_space.h`](ai_space.h.md) · [`magic_box3.h`](magic_box3.h.md) · [`moving_objects.h`](moving_objects.h.md) · [`GameObject.h`](GameObject.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`ai_obstacle.h`](ai_obstacle.h.md)
**Tier floor** — T2: dense geometry over an immutable grid; the only reason it is not higher is the per-frame cost of the bone scan.

## Purpose

The level graph ships prebuilt and knows nothing about creatures, crates or vehicles
standing on it. When a dynamic object is large enough to matter, the pathfinder must treat
the vertices under it as blocked. This file answers, for one object, "which vertices?" —
and answers it in a way that is cheap to *re-ask* when nothing changed, because the answer
is cached behind a checksum.

The hard part is not the grid scan, it is deciding the object's shape: a skinned model's
silhouette is the union of its bones' boxes, which is not itself a box.

## State

```text
RECORD AiObstacle
  object      : reference to the game object whose body this is
  actual      : bool             # is the cached area in step with the object's position?
  area        : list<int>        # blocked navigation vertex identifiers, ascending
  danger_area : list<int>        # declared, never filled
  crc         : int (32-bit)     # checksum of area; 0 when area is empty
  box         : 6 planes         # the fitted oriented box, as inward half-spaces
  min_box     : oriented box     # the same box in centre/axes/extent form

# Invariant: area is sorted ascending, because it is produced by walking the grid in
#   column-major order and the grid stores vertices sorted by packed column index.
#   Consumers rely on the order to merge obstacle areas cheaply.
# Invariant: crc is a function of area alone. Two obstacles with the same blocked set
#   have the same checksum, which is how a consumer detects "nothing changed" without
#   comparing lists.
```

## `compute_matrix` — fitting a box to a skinned body

**Contract** — Produces the transform of an oriented box enclosing every *visible* bone's
own box, inflated by a caller-supplied margin on each axis. Degenerate cases are handled
explicitly and each has a different answer. Allocates its scratch point set on the stack,
sized by the visible bone count — so an implausibly bony model is a stack risk, which is
the kind of thing a rebuild fixes for free.

```text
FUNCTION compute_matrix(additional_margin) -> transform
  IF no bones are visible
    RETURN a zero-scaled transform        # the object occupies nothing
  points = empty
  FOR EACH visible bone
    IF the bone's box has zero extent
      skip it, and stop counting it as visible   # attachment points, not body parts
      CONTINUE
    world = object transform composed with bone transform composed with box transform
    remember this bone's world transform and half-extent
    emit the bone box's 8 corners into points
  IF exactly one bone contributed
    RETURN that bone's transform scaled by its half-extent plus the margin
  fitted = minimum-volume oriented box over all points
  axes   = the three edge directions of the fitted box, normalized
  extent = half the fitted box's edge lengths, plus the margin per axis
  RETURN transform from the fitted centre, axes and extent
```

**Invariants** — Zero-extent bones must be excluded from the *count* as well as the point
set, or the later check that the point set is exactly eight points per visible bone fails.
The single-bone case is special-cased not as an optimization but because the
minimum-volume fit over eight coplanar-ish corners of one box is the box itself, and going
through the general path would rebuild it with numerical drift.

**Notes** — The minimum-volume oriented box fit is a general routine supplied elsewhere; it
is genuinely a fit, not an axis-aligned hull, because a stalker lying diagonally across the
grid would otherwise block a vastly larger rectangle than its body covers.

## `prepare_inside` — box to planes

**Contract** — Fits the box with a margin of half a navigation cell on every axis,
transforms the unit cube's eight corners through it, derives the axis-aligned extent of
those corners, and builds the box's six bounding planes from corner triples. Returns the
axis-aligned extent so the caller knows which part of the grid to scan.

**Notes** — The half-cell margin is the load-bearing constant: a navigation vertex is the
*centre* of a cell, and a body that overlaps a cell at all should block it, so the test
volume is inflated by half a cell in each direction. It is behind a flag that is compiled
on; with it off the obstacle is exactly the body, and objects clip the edges of cells they
stand on without blocking them.

## `inside` — is a point, or a vertex, inside the box

**Contract** — Three layered tests.

```text
FUNCTION inside(point, radius) -> bool
  # A point is inside when it is on the inner side of all six planes,
  # with the radius as slack.
  FOR EACH of the 6 planes
    IF signed distance from plane to point > radius THEN RETURN false
  RETURN true

FUNCTION inside(point, radius, increment, steps) -> bool
  # A vertical probe: the same test repeated up a column.
  FOR i IN 0 .. steps-1
    IF inside(point raised by i * increment, radius) THEN RETURN true
  RETURN false

FUNCTION inside(vertex_id) -> bool
  offset = half a navigation cell, less an epsilon
  # Five ground samples: the vertex's four cell corners and its centre. The corners
  # use the vertex's own ground plane to get their height, so a sloped cell is
  # sampled on its actual surface, not on a flat square.
  FOR EACH of the 5 sample points
    IF inside(sample, epsilon, 0.3, 6) THEN RETURN true
  RETURN false
```

**Invariants** — The vertical probe rises six steps of 0.3 metres — 1.5 metres above the
floor. That range is the decision: an obstacle is anything intersecting the volume a
standing creature occupies over a cell, not merely anything touching the floor. A table
whose legs miss every sample point but whose top is at waist height still blocks, and an
object suspended above 1.5 metres does not.

The corner offset is shortened by an epsilon so that adjacent cells do not both claim the
shared boundary.

## `compute_impl` — the grid scan

**Contract** — Recomputes the blocked vertex set and its checksum. Clears the previous
result first. Leaves the checksum at zero when nothing is blocked, which is the caller's
"no obstacle" signal.

```text
FUNCTION compute_impl()
  (min_point, max_point) = prepare_inside()
  clamp both to the level graph's own bounds      # the box may stick out past the level
  (x_min, z_min) = grid column of min_point
  (x_max, z_max) = grid column of max_point
  area = empty
  FOR EACH column (x, z) in that rectangle
    key = x * row_length + z
    # The graph stores vertices sorted by packed column key; several vertices can
    # share a column at different heights (a bridge over a path). Find the first by
    # binary search and walk the run.
    FOR EACH vertex in the run with this key
      IF inside(vertex) THEN append its identifier to area
  IF area is empty
    crc = 0
  ELSE
    crc = checksum over the area's identifiers
```

**Invariants** — Clamping to the level's bounds before converting to grid coordinates is
required, not defensive: the conversion is a division that would wrap on an out-of-range
coordinate and select a column on the opposite side of the level.

The scan is rectangle-over-columns rather than a query against the graph, because the
graph is a flat sorted array with no spatial index beyond its column ordering; the binary
search per column *is* the index.

**Notes** — Two assertions are commented out on either side of this routine — one checking
the object claims to be an obstacle at all, one checking the result is empty (which is
plainly the inverse of the intent). Neither is live, and the second looks like a debugging
leftover rather than a rule.

## `on_move`

**Contract** — Marks the cached area stale. The only invalidation path; see the inline
twin for what that misses.

## `distance_to`

**Contract** — Distance from a point to the object's origin, not to the fitted box. Callers
use it to rank which obstacles are worth considering at all, where origin distance is
close enough and far cheaper.
