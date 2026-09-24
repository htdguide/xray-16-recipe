# src/xrCDB/ISpatial.cpp

> The loose octree of moving objects — constant-time placement by radius and
> position, and the registration lifecycle that keeps every object in exactly one node.

**Needs** — [`ISpatial.h`](ISpatial.h.md) · [`xrCore/_fbox.h`](../xrCore/_fbox.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a tree of small nodes with a free list; the pooling is a throughput
choice, not a layout requirement.

## Purpose

The moving half of the collision world. Everything that can be somewhere and can move is
registered here, and the structure exists to make three operations cheap in this order of
importance: *move* (every object, every frame), *query* (thousands per frame), *insert and
remove* (on spawn and death).

## State

The eight octant directions, as unit-signed offsets, indexed so that octant `i` is the
direction whose x, y and z signs are bits 0, 1 and 2 of `i`. Everything else is in
[`ISpatial.h`](ISpatial.h.md).

## `SpatialBase.is_placed_correctly`

**Contract** — true when the object's bounding sphere still fits within the node it is
cached in. Pure test, no side effect.

```text
FUNCTION is_placed_correctly() -> bool
  slack <- node_radius - sphere.radius
  RETURN sphere.centre is within `slack` of node_centre on all three axes
```

**This is the placement invariant, and it is what makes moving cheap.** An object that moved
but still satisfies it needs no work at all — and because a node's test volume is twice its
placement volume, most small movements satisfy it. A negative slack (an object larger than
its node's looseness) makes the test fail always, which is correct: such an object belongs
higher up.

## `SpatialBase.register` · `unregister`

**Contract** — enter and leave the index. Both are idempotent: registering something already
registered does nothing, and unregistering something absent does nothing. Registering marks
the object's sector stale, because a newly placed object's sector is unknown.

**Invariants** — after registering, the object is in exactly one node's item list and its
cached node fields describe that node. After unregistering, its cached node is absent and its
sector is reset to *none*.

**Notes** — the base destructor unregisters. That is the one ordering rule an implementor
must respect: an object must leave the index before the memory holding its spatial record
goes away, or the index holds a dangling entry that the next query walks. The engine asserts
the same rule from the other side — see the runtime invariants in
[`SYSTEM-REQUIREMENTS.md`](../../SYSTEM-REQUIREMENTS.md#6-conformance).

## `SpatialBase.moved`

**Contract** — called when the object's bound changed. Marks the sector stale, and — only if
the placement invariant now fails — removes and reinserts.

```text
FUNCTION moved()
  IF not registered: RETURN            # not an error: objects move before they exist
  mark sector stale
  IF is_placed_correctly(): RETURN     # the common case: no structural work at all
  remove ; insert
```

**Remove-then-insert, not a search for the new node.** Both are constant time, so there is
nothing to save by being clever, and the reinsert goes through exactly the same placement
rule as a fresh insert — which means there is only one way an object can ever be placed. A
rebuild tempted to walk up from the current node to find the new one gains nothing and
introduces a second placement path that can disagree with the first.

## `SpatialBase.set_sector`

**Contract** — accepts a recomputed sector, but only if the object's sector is currently
marked stale, and only if the id offered is a real one. Clears the stale mark.

The gate is not an optimization: the sector is recomputed by whoever happens to touch the
object, and an object that has not moved must keep the sector it was given rather than
accept whatever the last caller guessed.

## `Index.initialize`

**Contract** — sizes the index to a world bounding box: the centre is the box's centre and
the half-size is the box's *largest* half-extent, so the index is always a cube. Creates the
root. Called once per level.

A cube, not the box, because the octant rule halves all three axes together; a non-cubic
root would produce non-cubic nodes whose looseness differs per axis and whose placement test
would need three slacks.

## `Index.insert`

**Contract** — places an object at the unique node its radius and position select. Takes the
index's lock. Constant time in the number of objects; logarithmic in the ratio of world size
to object size, which is bounded by the minimum node size.

```text
FUNCTION insert(object)
  LOCK index DURING
    IF the object's sphere does not fit the root's loose volume
      put it in the root and record the root's bounds as its node      # see below
      RETURN
    place(root, centre, bounds)

FUNCTION place(node, node_centre, node_half)
  IF node_half <= minimum_node_size                    # resolution floor reached
    put object here ; cache (node_centre, 2 * node_half) ; RETURN

  child_half <- node_half / 2
  IF object.sphere.radius < child_half                 # it fits a level down
    octant <- sign bits of (object.sphere.centre - node_centre)
    child_centre <- node_centre + direction(octant) * child_half
    create the child if absent
    place(child, child_centre, child_half)
  ELSE
    put object here ; cache (node_centre, 2 * node_half)
```

**Radius picks the depth; position picks the branch.** The descent stops at the first level
whose children are too small for the object, so an object's depth is a function of its size
alone. Nothing is compared against anything else, no node is ever split or merged because of
a count, and there is no rebalancing — which is exactly what makes insertion and removal
constant time and is the requirement the whole structure was built to.

**An object too large or too far for the root is parked at the root with the root's bounds
recorded as its node.** Its placement test then always fails, so every `moved` reinserts it,
and it rejoins the structure properly as soon as it fits. That is a deliberate
self-correction rather than a rejection: objects legitimately exist outside a level's
bounding box (in transit, or off-level) and must not be lost.

## `Index.remove`

**Contract** — takes the object out of its cached node and prunes every now-empty ancestor.
Takes the lock. Constant time plus the depth.

```text
FUNCTION remove(object)
  LOCK index DURING
    node <- object's cached node
    take the object out of node's item list
    WHILE node is empty AND node has a parent
      detach node from its parent ; recycle node ; node <- parent
```

Empty interior nodes are recycled rather than freed, onto a free list the index owns. That
is why insertion can create a child unconditionally: nodes churn constantly as objects move
across octant boundaries, and allocating one per crossing would be the dominant cost of the
structure.

## `Index.update`

**Contract** — in instrumented builds, runs the consistency check
([`ISpatial_verify.cpp`](ISpatial_verify.cpp.md)). In a shipping build it does nothing,
including not taking the lock. The parameter it takes — a number of nodes to process — is
vestigial: nothing reads it. This was presumably an incremental maintenance pass that no
longer exists.

## Notes

**Insertion and removal are locked; the three queries are not uniformly so.** Insert, remove
and the box, sphere, ray and frustum queries all take the same single index lock, so in
practice the dynamic index *is* serialized — unlike the static tree, which needs no lock at
all because it is immutable. That asymmetry is the honest summary of the module's
concurrency story: *static queries scale across threads, dynamic queries do not.* A rebuild
that needs concurrent dynamic queries has to make the structure immutable per frame — build
it once from the frame's object positions and query the snapshot — rather than locking
harder.

The insertion path writes the object being inserted into a field on the index and then
recurses, rather than passing it down. That is a mechanism; a rebuild passes it as a
parameter. It is worth naming because that field makes `insert` re-entrant-unsafe in a way
the lock happens to hide.
