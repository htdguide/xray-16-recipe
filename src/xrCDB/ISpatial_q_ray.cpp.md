# src/xrCDB/ISpatial_q_ray.cpp

> Finds the moving objects a ray passes through, by walking the octree with the
> same slab test the static tree uses.

**Needs** — [`ISpatial.h`](ISpatial.h.md) · [`xrCore/_fbox.h`](../xrCore/_fbox.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the box test is the same wide-float slab test as the static tree's,
with the same reliance on how infinities propagate.

## Purpose

The dynamic counterpart of [`xrCDB_ray.cpp`](xrCDB_ray.cpp.md). Every ray question the game
asks reaches both: the static tree for the world, this for everything in it. It returns
*candidates* — objects whose bounding sphere the ray meets — not hits; the caller then tests
each candidate's actual collision form.

## State

Stateless between calls.

```text
RECORD RayWalk
  ray      : (origin, direction, 1/direction)
  mask     : set<Kind>
  range    : real         # shrinks under the nearest-only policy
  range_sq : real
  out      : reference to the caller's candidate list
```

## `Index.ray`

**Contract** — appends to a caller-supplied list every registered object that matches the
kind mask and whose bounding sphere the ray meets within range. Clears the list first. Takes
the index's lock for the whole walk, so it does not run concurrently with an insertion, a
removal or another query.

**Invariants** — under the nearest-only policy the range shrinks as candidates are found, so
the list is *pruned* but not ordered — the last entry is not necessarily the nearest, and
earlier entries may be farther than the final range. A caller wanting the nearest must still
compare.

```text
FUNCTION ray(out, options, mask, origin, dir, range) -> ()
  LOCK index DURING
    clear out
    walk(root, centre, bounds)

FUNCTION walk(node, node_centre, node_half)
  IF the ray misses the node's loose box (centre +/- 2*node_half): RETURN
  IF the entry distance exceeds range: RETURN

  FOR EACH object IN node.items
    IF object.kinds does not contain every bit of mask: CONTINUE   # AND, not OR
    result, t <- intersect(object.sphere, ray, range)
    IF result is "origin inside" OR (it hit AND t < range)
      IF policy is nearest-only: range <- min(range, t) ; range_sq <- range^2
      append object to out
      IF policy is first-only: RETURN

  FOR EACH non-empty octant
    walk(child, child_centre, node_half / 2)
    IF policy is first-only AND out is non-empty: RETURN
```

**The kind mask is an AND here**, unlike the box and frustum walks where it is an OR. The ray
callers pass compound masks — "collideable *and* an obstacle" — and depend on the conjunctive
reading. See the trap noted in [`ISpatial.h`](ISpatial.h.md).

**The children are walked in octant order, not in ray order.** As with the static tree, a
nearest-only query would prune far harder if the eight children were visited in the order the
ray enters them; the slab test already computes the entry parameter, so the ordering is
cheap. This is the same missed optimization in both indexes, and a rebuild should fix it in
both.

**A ray whose origin is inside an object's sphere counts as a hit** regardless of the
intersection parameter, which is what lets a creature's own queries find things it is
standing inside.

## the box test

The node's loose box — centre plus and minus twice the node's half-size — is tested with
exactly the slab test described in [`xrCDB_ray.cpp`](xrCDB_ray.cpp.md), in both a wide and a
scalar form chosen once per query, with the same handling of infinities for axis-parallel
rays and the same split between "returns a parameter" and "returns a point" that forces the
range prune to be written twice. It is duplicated here rather than shared, which is a
copy-and-paste artifact: a rebuild should have one slab test and two callers.

## Notes

The traversal is specialized on the three policy bits before it starts, for the same reason
the static tree's is: no option may be tested inside a walk that runs at every node.
