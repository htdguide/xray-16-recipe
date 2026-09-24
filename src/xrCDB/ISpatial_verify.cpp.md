# src/xrCDB/ISpatial_verify.cpp

> Walks the whole octree counting nodes and objects, and checks the counts against
> the running totals the index maintains.

**Needs** — [`ISpatial.h`](ISpatial.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a tree walk and two comparisons.

## Purpose

The index maintains two running counters — how many nodes exist, how many objects are
registered — incremented and decremented at every insertion, removal, node creation and node
recycle. Those counters are the only cheap handle anything has on the index's health, and
they are only trustworthy if every path maintains them. This walks the structure and proves
they are.

## State

Stateless.

## `Index.verify`

**Contract** — recursively counts the nodes reachable from the root and the objects in their
item lists, and returns whether both match the maintained counters. Halts the program on
mismatch in instrumented builds. Does not take the lock — its only caller already holds it.

```text
FUNCTION verify() -> bool
  nodes, objects <- count over the whole tree from the root
  RETURN nodes == stats.node_count AND objects == stats.object_count
```

**What a mismatch actually means.** More objects walked than counted means a double
insertion — an object reachable from two nodes, which will be reported twice by every query
and freed once. Fewer means an object was removed without the counter following, which
usually means it was freed while still linked and the walk is reading released memory. Fewer
nodes walked than counted means a leaked node, which is harmless; more means a node reachable
from two parents, which is not.

**This is a structural check, not a placement check.** It does not verify that any object
actually fits the node it sits in — that invariant is tested per object at insertion time and
per move by the placement test in [`ISpatial.cpp`](ISpatial.cpp.md). A rebuild wanting a
complete audit should check both here, since the placement invariant is the one a subtle bug
in the octant arithmetic would break, and the counters would not notice.

## Notes

Compiled out of a shipping build, along with its only caller. That is the right shape for an
invariant this expensive — it is a full tree walk — but a rebuild should keep it reachable
from a console command rather than deleting it, because it is the only diagnosis available
for the class of bug it catches.
