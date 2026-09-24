# src/Layers/xrRender_R2/r2_R_sun_support.h

> The convex-volume geometry both suns and the rain map stand on: extending a view frustum
> into a caster volume, fitting a light cuboid against a chain of rays, clipping boxes
> against a frustum, and the frustum-extrema construction the trapezoidal warp needs.

**Needs** — [`r2.h`](r2.h.md) · [`xrRender/r_sun_cascades.h`](../xrRender/r_sun_cascades.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: plane and polygon arithmetic over small fixed sets; no device
contact, though it is written against a specific clip-space depth range.

## Purpose

A directional light has no frustum. Deciding what it must render means constructing a
volume, and this header holds the two constructions the two suns use — a general, slow one
that grows a convex hull toward the light and reports its planes, and a fixed, fast one
that fits a cuboid of known size against the view frustum's edge rays and chains one
cascade to the next.

It also holds the small pieces those constructions need: a homogeneous transform that
divides through by w, the tables that name a unit cube's corners and faces, a frustum type
that computes its own eight extremal points by intersecting plane triples, and a clipper
that finds the bounding box of a set of boxes after clipping them against a frustum.

Everything here is duplicated once per backend to sit on that backend's matrix library.
The duplication is mechanical and carries no decision; the recipe describes the operation
once.

## State

```text
# The unit cube of clip space, and the faces of a frustum over it.
corners  : eight points, each component -1 or +1 in x and y, and 0 or +1 in depth
facetable: six quads over those corners; entries 4 and 5 are the near and far faces

# Tuning constants used by both suns.
cop_offset        = 1200 m   # how far "behind" the camera a directional light's
                             # virtual viewpoint is placed
ortho_near_offset = 1000 m   # how far the orthographic near plane is pushed back so
                             # casters above the fitted hull still render
guaranteed_range  = 20 m     # the near band the legacy sun's refit may never exclude
```

**Invariants** — the depth component of `corners` is 0 at the near plane and +1 at the far
plane, which is one of the two clip-space conventions in use. A rebuild targeting a
convention with a depth range of −1 to +1 must change this table and nothing else; every
other construction in the file reads the table rather than assuming.

## `project`

**Contract** — transforms a point by a matrix and divides through by the resulting w.
Pure. Used everywhere a clip-space corner must be pulled back to world space through an
inverse transform.

**Notes** — the perspective divide is the point. Pulling the unit cube's corners back
through an inverse view-projection is how every construction here obtains the view
frustum's world-space corners, and that only works with the divide.

## `ConvexHull` — the general caster volume

**Contract** — a set of points and a set of polygons over them, with plane equations
computed on demand. Its one interesting operation extends the hull to infinity along a
direction and reports the resulting volume's planes, outward-facing.

**Invariants** — after every plane computation the polygons are re-oriented so that the
hull's centre of gravity classifies negative against all of them. A polygon whose first
three points are collinear falls back to its fourth, and a polygon that is degenerate in
both is *removed* rather than kept with a garbage normal — a hull that has lost a face is
recoverable, a hull with a meaningless plane is not.

```text
FUNCTION extend_toward_light(direction) -> list<plane>
  centre = the average of all points
  compute planes; flip any polygon the centre classifies positive against
  compute planes again
  FOR EACH polygon
      facing = does its normal point away from the light
      FOR EACH of its edges
          find or create the edge; add +1 if this polygon faces the light, -1 if not
      IF it faces away from the light, delete the polygon
  # the edges whose counters cancelled are interior; the rest are the silhouette
  FOR EACH edge whose counter is zero
      extrude its two endpoints along the light direction
      add the resulting quad as a new polygon
  re-orient and recompute; export every polygon's plane
```

**Notes** — this is the classical silhouette-extrusion used for shadow volumes, applied to
a frustum rather than to a mesh. The counter arithmetic is what identifies the silhouette:
an edge shared by two polygons that agree about the light cancels, an edge on the boundary
between a lit and an unlit polygon does not. The volume is deliberately *not capped* at
the far end — the light is at infinity, so the volume is a prism that never closes, and
the culling frustum built from its planes is correct without a cap.

The comment in the source calls the implementation naive and slow, which it is: it is
linear scans over vectors and repeated plane recomputation, running once per shadow region
per frame over a hull of eight points. That is affordable exactly because the hull is a
frustum. A rebuild should keep the algorithm and not bother optimizing it.

## `FixedCuboid` — the chained cascade fit

**Contract** — fits a light cuboid of known world size against the four edge rays of a
view-frustum slice, reports the culling planes, reports the translation the fit needed,
and advances the rays to where the fit ended so the next cascade can start there. Pure
geometry; writes back into its own ray list.

**Invariants** — every side plane of the cuboid must classify the light's reference point
positive on entry and on exit; the fit translates the light, not the cuboid's orientation.
The fit is skipped entirely when the view direction and the light direction are within
floating-point tolerance of parallel — the construction has no answer there, and the
caller falls back to the unfitted transform.

```text
FUNCTION fit(map_size, clip_at_view_near) -> (planes, translation, advanced rays)
  IF the view and light directions are parallel THEN RETURN nothing

  compute the cuboid's four side planes
  align_planes = up to two side planes the view direction points into
      # these are the planes the frustum can press against from behind

  # --- step 1: slide the cuboid until it just touches the rays ----------------
  FOR EACH align plane
      shift it by the smallest signed distance from any ray origin
  translate the light by the accumulated shift

  # --- step 2: push further if a ray leaves through the plane ------------------
  FOR EACH align plane
      FOR EACH ray whose direction points out through it
          compute how far the plane must move for the ray to stay inside,
              as a fraction of the distance already travelled
          keep the largest such fraction, which must not exceed one
      push the plane by that fraction of its current offset
  translate again; move the cuboid's planes to match

  # --- step 3: culling planes from the ray edges -------------------------------
  FOR EACH ray
      plane through the ray's origin, normal = ray direction x light direction
      IF every ray lies on one side of it, keep it, oriented outward
  IF clip_at_view_near AND the view is not nearly along the light
      add a plane through the view origin perpendicular to both, pushed out to the
      furthest ray origin — unless some ray advances past it, in which case the plane
      passes through the view origin exactly
  add all four cuboid side planes, inverted

  # --- step 4: advance the rays for the next cascade --------------------------
  FOR EACH ray
      distance = the nearest intersection with any side plane, capped at map_size
      move the ray's origin forward by that distance
```

**Notes** — step 4 is the chaining. Each ray is walked forward to where it exits this
cascade's cuboid, and the next cascade starts its fit from those advanced origins. That is
what makes the cascades cover contiguous, non-overlapping slices of the frustum without
computing split distances at all: the split is wherever the previous cuboid's wall is.

The cap at the map size, and the fallback of "just use the map size" for a ray that runs
nearly parallel to a plane, keep a grazing ray from advancing to infinity and collapsing
the next cascade.

Step 2's fraction is asserted not to exceed one. Exceeding it would mean the required push
is larger than the distance already travelled, which the construction cannot represent; it
happens only for configurations step 0 already excluded.

## `Frustum` — the extrema construction

**Contract** — built from a projection matrix; produces six normalized planes, a per-plane
index of the nearest corner (for fast box tests), and the eight points where plane triples
intersect. Pure.

```text
FUNCTION build(matrix)
  planes = fourth column ± each of the first three columns   # left/right/bottom/top/near/far
  normalize each
  FOR EACH plane, precompute which corner of a box is nearest it, as a bit triple
  FOR EACH of the eight combinations of (near|far, bottom|top, left|right)
      intersect those three planes
```

**Invariants** — three planes intersect in a point only when their triple product is not
near zero and not infinite; the intersection reports failure otherwise, which for a valid
projection never happens.

**Notes** — the eight extrema are what the trapezoidal warp in
[`render_phase_sun_old.cpp`](render_phase_sun_old.cpp.md) rotates into light space and
fits its box against. Deriving them from the planes rather than from the unit cube is what
makes the construction independent of the depth convention.

## `Clipper` — clipped box bounds

**Contract** — given a list of world-space boxes, a frustum and a transform, returns the
bounding box, in transformed space, of everything inside the frustum. A box fully inside
contributes all eight corners; a box straddling contributes the clipped endpoints of every
segment between each pair of its corners; a box outside contributes nothing.

```text
FUNCTION clipped_bounds(boxes, transform) -> box
  result = empty
  FOR EACH box
      SELECT its frustum classification
        outside -> skip
        inside  -> transform its eight corners into result
        partial -> FOR EACH ordered pair of its corners
                       clip the segment between them against every frustum plane
                       IF anything survives, transform both endpoints into result
  RETURN result
```

**Notes** — the partial case is quadratic in the corners, 56 segments per box, and the
comment in the source concedes as much. It is affordable because the receiver list it runs
over is the *coarse* structure recorded by the main walk — whole visuals, a few hundred
boxes — and only the legacy sun uses it. The cascaded sun does not clip at all, which is
part of why it is the faster path.

Clipping the box's full graph of segments rather than its twelve edges is not a mistake:
the goal is the bounding box of the box's intersection with the frustum, and the diagonals
carry corners of that intersection that the edges alone would miss.
