# src/xrGame/quadtree.h

> A fixed-depth quadtree over the horizontal plane, used to answer "what is near this point" for cover points, doors and moving objects.

**Needs** — [`quadtree_inline.h`](quadtree_inline.h.md) · [`xrEngine/profiler.h`](../xrEngine/profiler.h.md)
**Used by** — [`cover_manager.cpp`](cover_manager.cpp.md) · [`cover_manager.h`](cover_manager.h.md) · [`cover_manager_inline.h`](cover_manager_inline.h.md) · [`doors_manager.h`](doors_manager.h.md) · [`moving_objects.h`](moving_objects.h.md) · [`quadtree_inline.h`](quadtree_inline.h.md)
**Tier floor** — T1: a hand-rolled node pool with type-punned leaves and a fixed depth chosen against a cell-size budget

## Purpose

Three systems need the same question answered many times per frame: which **cover** points
lie within a radius of a creature, which doors are near a path, which moving objects a
creature might collide with. All three are *points on a level's floor* — the vertical
extent of a level is small compared to its footprint — so the index splits on two axes
only, and the third is ignored entirely.

The tree is the engine's own rather than the collision database's because the contents are
**dynamic** (objects move, cover is rebuilt) and small (thousands, not hundreds of
thousands), which is the opposite of what the static collision database is built for.

## State

```text
RECORD QuadTree<T>
  center      : vec3        # of the level's bounding box
  radius      : real        # half the longer horizontal side of that box
  max_depth   : int         # fixed; see below
  root        : node or none
  leaf_count  : int         # number of objects, not of leaves
  node_pool   : Pool<Node>
  item_pool   : Pool<ListItem>

RECORD Node
  children : list<4> of (node or list-head or none)   # quadrant order: (-x,-z) (-x,+z) (+x,-z) (+x,+z)

RECORD ListItem
  object : T
  next   : ListItem or none
```

**Invariants**

- **Every object sits at exactly `max_depth`.** The tree does not subdivide on demand and
  does not collapse; depth is a property of the whole tree, fixed at construction from the
  level's size and a requested minimum cell size:
  `max_depth = round(log2(2 * radius / min_cell_size))`. So a cell is at worst
  `min_cell_size` across, and the caller chooses the trade between tree height and cell
  occupancy by choosing that size.
- **A slot at `max_depth` holds a linked list of objects, not a child node.** The same
  pointer field means "child" above the bottom and "list head" at the bottom; there is no
  tag. Depth alone decides which it is, which is why depth is threaded through every
  operation. A rebuild should make this a tagged union or two distinct node kinds — the
  punning is an artifact of wanting one 4-pointer node, not a decision.
- **Quadrant order is frozen** because the neighbour-visiting logic in the radius query
  addresses quadrants by arithmetic on the index rather than by name.
- Insertion does not check for duplicates and removal asserts the object is present;
  the tree trusts its owner's bookkeeping.
- The tree indexes by the object's *current* position, read through a `position()` method
  the element type must supply. An object that moves must be removed and reinserted; the
  tree never re-reads a position it has already filed.

## Storage pools

**Contract** — nodes and list items are drawn from block-allocated free lists rather than
individually. A block is a fixed-count array; freed entries are threaded onto a free list
through the entry's own first pointer field, and a fresh block is allocated when the list
empties. The block count the caller states at construction is therefore a *hint*, not a
cap — exceeding it costs one more block, not a failure.

Clearing the tree drops every block at once rather than walking the tree, which is the
reason the pools exist: rebuilding cover for a level is a whole-tree operation.

**Notes** — the free-list entries are zeroed on hand-out, so a reused node starts with no
children. That zeroing is load-bearing; without it a recycled node inherits a stale
quadrant.

## `insert` · `remove` · `find` · `nearest` · `all` · `clear` · `size` · `empty`

Contracts and algorithms are in [`quadtree_inline.h`](quadtree_inline.h.md).
