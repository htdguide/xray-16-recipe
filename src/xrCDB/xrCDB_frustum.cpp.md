# src/xrCDB/xrCDB_frustum.cpp

> Collects every static triangle inside a convex volume of half-spaces, exactly or
> conservatively.

**Needs** — [`xrCDB.h`](xrCDB.h.md) · [`Frustum.h`](Frustum.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: plane arithmetic over small vectors, nothing device- or layout-facing.

## Purpose

The frustum query: give it any convex region expressed as a set of inward half-spaces and
it returns the static triangles inside. It is the most general of the three queries — a box
query is a special case of it, and [`xr_area_query.cpp`](xr_area_query.cpp.md) uses exactly
that to answer *oriented*-box questions, which the axis-aligned box query cannot.

Its callers are the ones that need geometry shaped to a view rather than to a ball: light
and shadow geometry gathering, the level editor's selection, and anything projecting onto
the world.

## State

Stateless between calls.

```text
RECORD FrustumTraversal
  results : reference to the caller's result buffer
  verts   : reference to the model's vertices      # borrowed
  tris    : reference to the model's triangles     # borrowed
  frustum : reference to a Frustum                 # borrowed; see Frustum.cpp
```

## `Collider.frustum_query`

**Contract** — fills the collider's result buffer with the triangles inside the frustum.
Waits for the model, clears the buffer, descends. Concurrency as for every query.

```text
FUNCTION frustum_query(model, frustum, options) -> ()
  AWAIT model ready
  clear results
  descend(model.tree.root, active = frustum.all_planes_mask, policy = options)

FUNCTION descend(node, active)
  visibility, active' <- frustum.classify_box(node.box, active)
  IF visibility == outside: RETURN

  IF node.pos is a leaf: test_triangle(node.pos.triangle)
  ELSE                 : descend(node.pos, active')
  IF policy is first-only AND results is non-empty: RETURN
  IF node.neg is a leaf: test_triangle(node.neg.triangle)
  ELSE                 : descend(node.neg, active')
```

**The active-plane mask is the whole optimization of this traversal.** A node's box is
classified against only the planes still marked active; a plane the box lies entirely
inside cannot reject anything below that node, so it is cleared from the mask handed to the
children. Deep in the tree, most nodes are tested against one or two planes instead of six
or twelve. The mask is passed *by value* down the recursion, never by reference: clearing a
plane must affect one subtree, not the siblings visited afterwards. A rebuild that shares
the mask by mistake will silently admit geometry outside the frustum, and the bug will
look like a rendering artifact rather than a collision one.

### the leaf test

Under the conservative policy, reaching a leaf is acceptance: the triangle is reported
because its node's box overlapped the frustum. Under **full-test**, the triangle is clipped
against every frustum plane in turn (the same clipping routine the frustum type exposes,
see [`Frustum.cpp`](Frustum.cpp.md)) and is reported only if a non-degenerate polygon
survives. The clipped polygon itself is discarded — only the original triangle's three
vertices go into the result — so the exact path buys precision, not geometry.

### the result policy

As with the box query, hits carry no distance, so the policies are *all* and *first-only*,
and the unbounded case really is unbounded.

## Notes

This is the only query where the conservative policy differs from the exact one by a large
margin rather than a corner case: a frustum box test accepts an entire node whose box
merely touches the volume, so a shallow frustum over a dense level returns a great deal of
geometry that is nowhere near it. Callers that want the geometry rather than a yes/no must
ask for the full test.
