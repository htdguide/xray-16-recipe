# src/xrCDB/ISpatial_q_frustum.cpp

> Finds the moving objects inside a convex volume — the query the renderer's
> visibility pass and every shadow-casting light are built on.

**Needs** — [`ISpatial.h`](ISpatial.h.md) · [`Frustum.h`](Frustum.h.md) · [`xrCore/_fbox.h`](../xrCore/_fbox.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: plane tests down a tree.

## Purpose

*What is visible from here.* The renderer collects the frame's renderable objects with this,
once per camera and once per shadow-casting light per frame, so it runs more often than any
other dynamic query and over the whole index rather than a small region.

## State

Stateless between calls.

## `Index.frustum`

**Contract** — appends to a caller-supplied list every registered object that matches the
kind mask and whose bounding sphere is not wholly outside the frustum. Clears the list
first. Takes the index's lock for the whole walk.

```text
FUNCTION frustum(out, options, mask, frustum) -> ()
  LOCK index DURING
    clear out
    walk(root, index.centre, index.bounds, frustum.all_planes_mask)

FUNCTION walk(node, node_centre, node_half, active)
  visibility, active' <- frustum.classify_box(node's loose box, active)
  IF visibility == outside: RETURN

  FOR EACH object IN node.items
    IF object.kinds shares no bit with mask: CONTINUE
    per_object <- active'                       # a COPY: the sphere test consumes it
    IF frustum.classify_sphere(object.sphere, per_object) == outside: CONTINUE
    append object to out

  FOR EACH non-empty octant
    walk(child, child_centre, node_half / 2, active')
```

**Two levels of active-plane mask, and the copy between them is load-bearing.** The node's
box test narrows the mask for the subtree; each object's sphere test then narrows a
*private copy* of that, because a plane cleared by one object's sphere says nothing about
the next object. Passing the subtree mask into the sphere test by reference would silently
admit objects outside the frustum — the same trap as in
[`xrCDB_frustum.cpp`](xrCDB_frustum.cpp.md), one level deeper.

**No result policy at all.** Unlike the ray and box walks, this one has no first-only and no
early exit: it always collects everything. That is right for its callers — a render pass
wants the whole visible set — and it means the query is unbounded by construction. Nothing
caps how many objects a frame may collect.

## Notes

This is the one dynamic query whose cost is dominated by the *breadth* of the walk rather
than by the tests: a camera frustum reaches most of the level, so most of the octree is
visited every frame. That is the argument for the active-plane mask, which is what keeps the
per-node cost at one or two plane evaluations instead of six.
