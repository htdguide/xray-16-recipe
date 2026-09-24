# src/xrGame/quadtree_inline.h

> The quadtree's operations: descend by quadrant to a fixed depth, and a radius query that visits only the quadrants a circle actually overlaps.

**Needs** — [`quadtree.h`](quadtree.h.md)
**Used by** — [`quadtree.h`](quadtree.h.md)
**Tier floor** — T1: pooled nodes, depth-punned leaf slots, and a hand-specialised overlap test

## Purpose

Implements the tree declared in [`quadtree.h`](quadtree.h.md), which carries the state and
the invariants. Everything below assumes them — in particular that objects live only at
`max_depth` and that the slot at that depth is a list head.

## Construction

**Contract** — takes the level's bounding box, a minimum cell size, and pool block sizes.
The tree's radius is **half the longer of the two horizontal extents**, so the tree is
square and covers the box; the centre is the box's centre. Depth is derived from the cell
size (see the declaration twin). Requires a non-zero cell size strictly smaller than the
radius — a tree of depth zero or less has no quadrants to descend.

## `neighbour_index` — the descent step

**Contract** — given a position and the current cell's centre and half-extent, returns
which of the four quadrants the position falls in **and moves the centre into that
quadrant**. The two are done together because every caller needs both, and doing them apart
is where an implementation drifts out of sync with the frozen quadrant order.

```text
FUNCTION descend(position, centre INOUT, half_extent) -> int
  # ties (exactly on a boundary) go to the low side on both axes
  IF position.x <= centre.x THEN
    IF position.z <= centre.z THEN centre -= (h,0,h) ; RETURN 0
    ELSE                          centre += (-h,0,h) ; RETURN 1
  ELSE
    IF position.z <= centre.z THEN centre += (h,0,-h) ; RETURN 2
    ELSE                          centre += (h,0,h)  ; RETURN 3
```

## `insert`

**Contract** — files an object at the leaf its position falls in, creating interior nodes
on the way down. Never fails; never subdivides beyond `max_depth`. Constant-ish cost:
`max_depth` steps.

```text
FUNCTION insert(object)
  centre = tree.centre ; half = tree.radius ; slot = ref to root
  FOR depth = 0 ...
    IF depth == max_depth THEN
      item = item_pool.take()
      item.object = object
      item.next   = slot as list-head        # prepend; order within a leaf is not meaningful
      slot = item
      leaf_count += 1
      RETURN
    IF slot is none THEN slot = node_pool.take()
    half = half / 2
    slot = ref to slot.children[descend(object.position, centre, half)]
```

## `remove`

**Contract** — unlinks an object from its leaf and returns it, then prunes interior nodes
that were left with no children. Asserts the object is present: removing something the tree
never held is a bookkeeping error in the caller.

```text
FUNCTION remove(object, slot INOUT, centre, half, depth) -> T
  IF depth == max_depth THEN
    unlink the item whose object is `object` from the list at `slot`
    # the list head lives in `slot` itself, so removing the first item rewrites the slot
    item_pool.give_back(item) ; leaf_count -= 1 ; RETURN object
    FAIL WITH not-present

  half = half / 2
  index  = descend(object.position, centre, half)
  result = remove(object, slot.children[index], centre, half, depth+1)
  IF every child of slot is none THEN node_pool.give_back(slot)
  RETURN result
```

**Notes** — pruning is checked on the way *up*, one level at a time, so an emptied subtree
collapses fully in a single removal.

## `find`

**Contract** — returns the object stored at (approximately) a position, or nothing. Walks
to the leaf and then scans the leaf's list for an object whose position is *similar* to the
query — an epsilon comparison, not equality, because the caller's position has usually made
a round trip through a float transform. Returns the first match, so a leaf holding two
coincident objects answers arbitrarily.

## `nearest` — the radius query

**Contract** — appends every stored object within `radius` of a position (measured in the
horizontal plane only) to a caller-supplied list, optionally clearing it first. The
recursion prunes by quadrant overlap rather than by distance to a node's bounding box, and
it is written as an explicit case analysis rather than as four child tests.

```text
FUNCTION nearest(position, radius, out, node, centre, half, depth)
  IF node is none THEN RETURN
  IF depth == max_depth THEN
    FOR EACH item IN list at node
      IF horizontal_distance_squared(item.object.position, position) <= radius^2 THEN
        out.append(item.object)
    RETURN

  half = half / 2
  index = descend(position, centre_copy, half)      # the quadrant the query point is in

  near_z = |position.z - centre.z| < radius         # circle crosses the horizontal split
  near_x = |position.x - centre.x| < radius         # circle crosses the vertical split

  IF near_z AND near_x AND (dx^2 + dz^2 < radius^2) THEN
    recurse into ALL FOUR quadrants                 # the circle covers the centre point
    RETURN

  recurse into quadrant `index`                     # always: the query point's own cell
  IF near_z THEN recurse into the quadrant across the horizontal split
  IF near_x THEN recurse into the quadrant across the vertical split
  # the diagonal quadrant is reached only through the all-four case above
```

**Invariants** — the query is **horizontal**: the vertical coordinate is not compared at
all, so a cover point directly above the query passes. That is correct for the users (floor
points on one level) and would be wrong for a general spatial index.

**Notes**

- The three-way structure is exactly right and easy to get wrong: visiting the diagonal
  quadrant requires the circle to contain the cell's *corner*, which is what the
  `dx² + dz² < r²` test checks. Skipping that test and visiting all four whenever both axes
  are near would be correct but slower; visiting only the two axis-adjacent quadrants
  without it would **miss objects** near the corner.
- The results list is appended to, not sorted, and no budget is enforced. A caller asking
  for a large radius on a dense tree gets everything.

## `all`

**Contract** — appends every object in the tree to a list, in quadrant order. Used when a
whole layer is being rebuilt or debug-drawn.

## `clear` · `size` · `empty`

**Contract** — clearing drops both pools' blocks wholesale and resets the root and the
count; it does not walk the tree and does not destroy the objects, which the tree never
owned. `size` is the object count, not the node count.
