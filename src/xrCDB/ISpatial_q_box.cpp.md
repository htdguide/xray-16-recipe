# src/xrCDB/ISpatial_q_box.cpp

> Finds the moving objects overlapping an axis-aligned box — and, through it, the
> ones overlapping a sphere.

**Needs** — [`ISpatial.h`](ISpatial.h.md) · [`xrCore/_fbox.h`](../xrCore/_fbox.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: box overlap tests down a tree.

## Purpose

The most-used dynamic query: *what is near this point*. Proximity checks, area-of-effect,
the AI's "who is around me", and the nearest-objects helper in
[`xr_area.cpp`](xr_area.cpp.md) all come through here.

## State

Stateless between calls.

## `Index.box`

**Contract** — appends to a caller-supplied list every registered object that matches the
kind mask and whose bounding sphere's *box* overlaps the query box. Clears the list first.
Takes the index's lock for the whole walk.

```text
FUNCTION box(out, options, mask, centre, half_sizes) -> ()
  LOCK index DURING
    clear out
    walk(root, index.centre, index.bounds)

FUNCTION walk(node, node_centre, node_half)
  IF the query box and the node's loose box are disjoint: RETURN

  FOR EACH object IN node.items
    IF object.kinds shares no bit with mask: CONTINUE          # OR, not AND
    IF the box around object.sphere is disjoint from the query box: CONTINUE
    append object to out
    IF policy is first-only: RETURN

  FOR EACH non-empty octant
    walk(child, child_centre, node_half / 2)
    IF policy is first-only AND out is non-empty: RETURN
```

**The object test is box-against-box, not box-against-sphere.** The object's sphere is
converted to its enclosing box and compared corner-wise. That over-admits at the corners —
an object whose sphere misses the query box can have a box that does not — which is why
[`xr_area.cpp`](xr_area.cpp.md)'s nearest-objects helper runs an exact sphere test over the
results. The looseness is deliberate: the box test is six comparisons, the sphere test is a
distance, and most callers filter anyway.

**The kind mask is an OR here.** Any overlap with the mask admits the object. This is the
opposite of the ray query's reading; see [`ISpatial.h`](ISpatial.h.md).

**Nearest-only is not offered**, because a box query has no distance to be nearest by. The
policies are *all* and *first-only*.

## `Index.sphere`

**Contract** — the box query over the sphere's enclosing cube, with no further filtering.
So it is *conservative*: it returns everything in the cube, including objects in the cube's
corners that the sphere misses. The name promises more than it delivers and every caller
that needs the exact answer filters afterwards — which most do, so the sloppy version is the
one that is cheap and the exact one is the caller's business. A rebuild should name it for
what it is.
