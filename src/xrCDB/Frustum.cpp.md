# src/xrCDB/Frustum.cpp

> A convex volume as a small set of inward half-spaces, with the classification
> and clipping operations the visibility pass and the collision queries both live on.

**Needs** — [`Frustum.h`](Frustum.h.md) · [`xrCore/_plane.h`](../xrCore/_plane.h.md) · [`xrCore/FixedVector.h`](../xrCore/FixedVector.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: plane and matrix arithmetic over small vectors, with a fixed-capacity
polygon buffer that wants to be a stack value rather than a heap allocation.

## Purpose

Everything in this engine that asks "what can be seen from here" asks it as a frustum: the
camera, every shadow-casting light, every portal the visibility walk steps through, every
occluder, and the oriented-box collision query. This file is the type they all share.

It lives in the collision module because the collision module's frustum query needs it, but
the renderer is by far its heavier user — the sector/portal visibility walk builds a new
frustum per portal per frame. A rebuild may well put it in the math layer instead.

## State

```text
RECORD Frustum
  planes : list<CachedPlane>     # at most 12
  count  : int

RECORD CachedPlane
  normal  : vec3                 # unit length; points OUT of the volume
  d       : real
  corner  : int (0..7)           # which box corner is "most outward" for this normal

# Invariants
#   normal is unit length -- every construction path normalizes, because classify()'s
#     result is compared against a radius and against a tolerance, both in world units
#   corner is derived from the signs of normal's three components and must be recomputed
#     whenever the plane changes
#   count <= 12
```

**The sign convention is the one thing to get right.** `classify(p) = normal . p + d`, and a
point is **inside** the volume when the result is **non-positive** for every plane. Outward
normals with an "inside means negative" test is the opposite of the more common convention,
and every test and every clip in this file depends on it.

**Twelve planes, not six.** A camera frustum has six. A portal frustum has as many planes as
the portal polygon has edges, plus a near and a far plane, and an occluder frustum has one
per silhouette edge. Twelve is the cap that keeps a frustum a fixed-size value that can sit
on the stack and be copied freely — which matters because the visibility walk creates one
per portal and discards it immediately. A polygon that would need more planes is simplified
first (below) rather than growing the frustum.

The polygon buffer used for clipping is likewise fixed-capacity — four times the plane
cap — for the same reason: clipping happens per triangle inside a query traversal and must
not allocate.

## `Frustum.classify_box`

**Contract** — classifies an axis-aligned box against the frustum as *outside*, *partially
inside* or *fully inside*, given and returning a mask of which planes still matter. Never
allocates. The dominant operation in the engine's visibility pass.

```text
FUNCTION classify_box(box, active) -> (visibility, active')
  FOR EACH plane i WHERE active has bit i
    r <- classify_box_against_plane(planes[i], box)
    IF r == fully_inside : clear bit i from active   # cannot reject anything below here
    ELSE IF r == outside : RETURN (outside, empty)
  RETURN (active is empty ? fully_inside : partial, active)
```

**The mask is a decision, not a micro-optimization.** Passing it down a hierarchy (the
collision tree in [`xrCDB_frustum.cpp`](xrCDB_frustum.cpp.md), the spatial octree in
[`ISpatial_q_frustum.cpp`](ISpatial_q_frustum.cpp.md), the renderer's sector walk) means a
node deep in the tree is usually tested against one or two planes rather than six or
twelve. The mask must be passed **by value**: a plane cleared while descending one child
must be active again for the sibling.

### the corner trick

Testing a box against a plane needs only two of the box's eight corners: the one farthest
along the normal and the one farthest against it. Which two depends only on the *signs* of
the normal's three components — eight possibilities — so each plane caches that choice when
it is built, as an index into a fixed eight-by-six table of which of the box's six
coordinates to take for each of the two corners.

```text
FUNCTION classify_box_against_plane(plane, box) -> visibility
  near_corner <- the box corner farthest AGAINST plane.normal   # by plane.corner
  IF classify(near_corner) > 0: RETURN outside        # whole box is beyond the plane
  far_corner  <- the box corner farthest ALONG plane.normal
  IF classify(far_corner) <= 0: RETURN fully_inside   # whole box is behind the plane
  RETURN partial
```

Two plane evaluations instead of eight. The table itself is mechanical — for each sign
combination it lists the three coordinate choices for each corner — and a rebuild can
compute it rather than write it out.

## `Frustum.classify_sphere`

**Contract** — the same three-way classification for a sphere, with the same active-plane
mask. A sphere whose centre is farther than its radius beyond a plane is outside; a sphere
entirely behind a plane clears that plane from the mask.

There is also a **dirty** variant that answers only *not outside* — no mask, no
fully-inside distinction, unrolled over the plane count. It is for the case where the
caller will do exact work anyway and only wants a trivial reject; a rebuild that does not
care about the last few percent can express it as the general version with the results
thrown away.

## `Frustum.classify_sphere_and_box`

**Contract** — classifies a sphere and its box together against the same plane set, using
the sphere test first and falling back to the box test only for the planes the sphere
straddles. It exists because objects in the scene carry both bounds and the sphere test is
cheaper but looser: a long thin object's sphere straddles planes its box clears. Taking the
cheap answer when it is decisive and the tight answer when it is not is strictly better
than either alone, for one extra branch.

## `Frustum.clip_polygon`

**Contract** — clips a convex polygon by every plane in turn and returns the surviving
polygon, or nothing if it degenerates below three vertices. Works in two caller-supplied
buffers, ping-ponging between them, so it allocates nothing. Both buffers are written.

```text
FUNCTION clip_polygon(a, b) -> optional<polygon>
  src <- b ; dst <- a
  FOR EACH plane P
    swap(src, dst) ; clear dst
    classify every vertex of src against P
    close src by appending its first vertex again
    FOR EACH consecutive pair (u, w) in src
      IF u and w are the same point within tolerance: CONTINUE
      IF u is inside
        emit u
        IF w is outside: emit the crossing point of segment (u,w) with P
      ELSE IF w is inside
        emit the crossing point of segment (u,w) with P
    IF count(dst) < 3: RETURN none         # fully clipped away
  RETURN dst
```

This is Sutherland–Hodgman, with two details worth carrying:

- **Coincident consecutive vertices are skipped.** A clipped polygon routinely produces
  duplicate vertices at a corner, and feeding a zero-length edge into the next plane's
  crossing computation divides by zero.
- **A crossing whose denominator is zero is dropped rather than clamped.** The segment is
  parallel to the plane, so there is no crossing to emit, and the two endpoints have
  already been handled by the inside tests.

The caller decides what "the polygon survived" means: the frustum query uses it as an exact
triangle-inside-frustum predicate and discards the clipped result; the portal construction
below uses the clipped polygon itself.

## `Frustum.classify_polygon_dirty`

**Contract** — true when *every* vertex of a polygon is inside *every* plane. Not the same
question as clipping: this asks whether the polygon is wholly contained, not whether it
overlaps. Used where a cheap conservative containment answer is enough.

## `Frustum.from_matrix`

**Contract** — extracts up to six planes from a combined view-projection matrix, selecting
which of left/right/top/bottom/near/far to include by a mask. Normalizes every plane and
caches its corner index.

Each plane is a sum or difference of the matrix's fourth column with one of the first
three, negated to match this file's outward convention. Selecting a subset is not a
convenience: a shadow frustum built from a directional light wants the four side planes
without the near and far, and the visibility walk builds camera frustums with the far plane
omitted so a distant sector is not clipped away before its own near/far are applied.

The extracted planes are *not* unit length as they come out of the matrix, and every use in
this file compares a classification against a world-space distance, so the normalization is
required rather than tidy.

## `Frustum.from_points`

**Contract** — builds a pyramid: given a convex polygon and an apex, one plane per polygon
edge, each through the apex and that edge. Does not add a near or far plane — the caller
adds those if it wants them. Requires at least three points and fewer than the plane cap.

Plane construction from three points uses the *precise* variant, which is the numerically
careful one, because a portal polygon's edges can be very short relative to the level's
coordinate range and a sloppy normal there tilts a whole sector's visibility.

## `Frustum.from_portal`

**Contract** — builds the frustum through a portal, as seen from a viewpoint: a pyramid
from the viewpoint through the portal polygon, plus the portal's own plane as the near
plane and the view's far plane. This is the operation the sector/portal visibility walk is
built on.

```text
FUNCTION from_portal(poly, viewpoint, view_projection) -> ()
  P <- the plane of poly
  IF count(poly) > 6                 # too many edges to fit the plane budget
    poly <- simplify_to_quad(poly, P)
    P    <- the plane of the simplified poly
  IF viewpoint is on the negative side of P
    reverse poly                     # so the pyramid's planes come out facing outward
    P <- the plane of the reversed poly
  from_points(poly, viewpoint)
  add P                              # near plane: nothing behind the portal is visible
  add the far plane extracted from view_projection
```

**The winding fix is the load-bearing step.** A portal is a two-sided surface stored with
one winding, and the pyramid's plane normals come out inverted if the viewer is on the
other side. Reversing the polygon rather than negating the planes keeps the polygon and its
plane consistent for the near-plane step that follows.

**Six edges is the budget.** Six edge planes plus near plus far is eight, comfortably under
the cap, and it leaves room for the clipped-portal case where a frustum is rebuilt from an
already-clipped polygon. A portal with more edges is simplified rather than refused.

## `Frustum.simplify_to_quad`

**Contract** — replaces a polygon with the four corners of its bounding rectangle *in its
own plane*. Conservative: the result contains the original, so the frustum through it
admits everything the true portal admits and a little more.

```text
FUNCTION simplify_to_quad(poly, plane) -> polygon
  build a frame whose forward axis is plane.normal, anchored at poly[0]
    # the up reference is the world vertical, unless the plane is nearly horizontal,
    # in which case use a horizontal axis instead -- otherwise the cross product degenerates
  project every vertex into that frame's 2D
  take the 2D bounding rectangle
  return its four corners projected back out
```

Admitting slightly too much is the right failure direction for visibility: a sector wrongly
admitted is drawn and looks correct, a sector wrongly rejected is a hole in the world.

## `Frustum.from_occluder`

**Contract** — builds the volume *behind* a convex occluder polygon as seen from a
viewpoint — the shadow of that polygon — while omitting the planes for edges that lie flat
on a clipping frustum's own planes.

```text
FUNCTION from_occluder(poly, viewpoint, clip) -> ()
  open <- all false, one per edge of poly
  FOR EACH plane P of clip
    FOR EACH edge (j, j+1) of poly
      IF both endpoints lie on P within tolerance: open[j] <- true
  clear
  add the plane of poly                            # the occluder's own face
  FOR EACH edge (j, j+1) of poly WHERE NOT open[j]
    add the plane through (viewpoint, poly[j], poly[j+1])
```

**An "open" edge is one created by clipping, not by the occluder.** When an occluder polygon
has been clipped to the view frustum, the clip introduces edges that are artefacts of the
view, not silhouettes of the occluder — and a plane erected on such an edge would wrongly
narrow the shadow volume. Detecting them as "both endpoints lie on a clip plane" is the
available test, and it is why this function needs the clipping frustum rather than just the
polygon.

## `Frustum.from_clipped_polygon`

**Contract** — clips a polygon by another frustum and, if anything survives, builds a
pyramid from the survivor and a viewpoint. Returns whether it succeeded. This is the
composition the visibility walk performs at every nested portal: the portal polygon is
clipped to the frustum that reached it, and the clipped result becomes the next frustum.

## `Frustum.from_planes`

**Contract** — builds a frustum directly from a list of planes, normalizing each and
caching its corner index. The escape hatch for callers with their own geometry — the
oriented-box query in [`xr_area_query.cpp`](xr_area_query.cpp.md) builds six planes from a
box's axes and hands them straight over.

## `Frustum.all_planes_mask`

**Contract** — the active-plane mask with every plane set. The starting value for every
hierarchical descent.

## Notes

`add` and every construction path recompute the corner index. A rebuild that lets a caller
mutate a plane in place must recompute it too; a stale corner index silently produces wrong
visibility for boxes, in a way that looks like a geometry bug.
