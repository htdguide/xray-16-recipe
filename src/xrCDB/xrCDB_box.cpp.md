# src/xrCDB/xrCDB_box.cpp

> Collects every static triangle that overlaps an axis-aligned box, either
> conservatively or exactly.

**Needs** — [`xrCDB.h`](xrCDB.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it is arithmetic over small fixed-size vectors with no layout or device
concerns; only the shared frame budget argues for going lower.

## Purpose

The box query. Its callers want *the geometry in a region*: the physics engine collecting
the static triangles near a moving body, footstep and material probes, and anything that
needs to enumerate a patch of world rather than pierce it.

It is separate from the ray query because the descent test is different (box against box,
not ray against box) and because the exact leaf test — a triangle against a box — is a
different and considerably more expensive predicate.

## State

Stateless between calls.

```text
RECORD BoxTraversal
  results : reference to the caller's result buffer
  verts   : reference to the model's vertices      # borrowed
  tris    : reference to the model's triangles     # borrowed
  center  : vec3
  extents : vec3                                   # half-sizes
  min,max : vec3                                   # the same box as corners, for the
                                                   # node test, which wants it that way
```

## `Collider.box_query`

**Contract** — fills the collider's result buffer with the triangles overlapping the given
axis-aligned box, described by its centre and half-sizes. Waits for the model, clears the
buffer, descends. Concurrency as for every query: the tree is read-only, the buffer is the
caller's.

**Invariants** — under the conservative policy, every triangle whose *leaf was reached* is
reported, so the result is a superset of the true overlap set. Under the exact policy the
result is exactly the overlap set. Both are supersets of nothing else: no overlapping
triangle is ever missed.

```text
FUNCTION box_query(model, center, half_sizes, options) -> ()
  AWAIT model ready
  clear results
  descend(model.tree.root, policy = options)

FUNCTION descend(node)
  IF the query box and node's box are disjoint on any axis: RETURN

  IF node.pos is a leaf: test_triangle(node.pos.triangle)
  ELSE                 : descend(node.pos)
  IF policy is first-only AND results is non-empty: RETURN
  IF node.neg is a leaf: test_triangle(node.neg.triangle)
  ELSE                 : descend(node.neg)
```

### the leaf test

The exact predicate is the separating-axis test between a box and a triangle, run in the
box's frame — translate the triangle so the box is centred at the origin, then look for a
separating axis among thirteen candidates, in three classes:

```text
FUNCTION triangle_overlaps_box(v0, v1, v2) -> bool     # already box-relative
  # class I -- the three box axes
  FOR EACH axis IN {x, y, z}
    IF min(v0,v1,v2)[axis] > extents[axis]: RETURN false
    IF max(v0,v1,v2)[axis] < -extents[axis]: RETURN false

  # class II -- the triangle's plane
  normal <- (v1 - v0) cross (v2 - v1)
  d      <- -(normal dot v0)
  IF the box lies entirely on one side of that plane: RETURN false

  # class III -- the nine cross products of a box axis with a triangle edge
  IF policy is exact:
    FOR EACH edge IN {v1-v0, v2-v1, v0-v2}
      FOR EACH axis IN {x, y, z}
        a <- axis cross edge
        IF the triangle's span along a and the box's span along a are disjoint:
          RETURN false
  RETURN true
```

**The point of the three classes is that they may be stopped early.** Class I rejects the
overwhelming majority of triangles for the cost of three interval comparisons; class II
catches the rest of the easy cases; class III is nine more axis tests and is what the
**full-test** option turns on. Skipping class III makes the query conservative: it admits
triangles that pass the box's axes and the triangle's plane but are in fact separated by an
edge-cross axis, which for real geometry is a small overcount of triangles near a corner of
the box. Callers that then run their own exact test — the physics bridge does — ask for the
cheap version deliberately.

Class III's nine tests are written with the constant component of each cross product folded
away, since a box axis has two zero components; the interval half-width reduces to a
two-term sum of the box's extents. That reduction is arithmetic, not a decision, but it is
worth reproducing: done naively, class III roughly triples the cost of the leaf test.

### the result policy

Box results carry no distance, so **nearest-only is meaningless here and is not offered**.
The policies are *all* (the default, and genuinely unbounded — a box over a big region can
return tens of thousands of triangles) and *first-only* (stop at one, used to answer "is
this volume empty"). A caller that cannot afford the unbounded case must shrink the box;
there is no budget parameter.

## Notes

Every reported hit copies the three vertex positions and the triangle's payload word into
the result, the same record the ray query fills, with the distance and barycentric fields
simply left as they were. A rebuild with a tagged result type should make that explicit
rather than leaving stale fields readable.
