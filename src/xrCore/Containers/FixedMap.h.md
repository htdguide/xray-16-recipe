# src/xrCore/Containers/FixedMap.h

> An unbalanced binary search tree whose nodes live in one growable array, built and thrown away every frame without ever freeing memory.

**Needs** — [`../../xrCommon/xr_allocator.h`](../../xrCommon/xr_allocator.h.md) · [`../../xrCommon/xr_vector.h`](../../xrCommon/xr_vector.h.md)
**Used by** — [`r__dsgraph_types.h`](../../Layers/xrRender/r__dsgraph_types.h.md)
**Tier floor** — T1: the tree's links are raw addresses into its own storage and are *rewritten* when that storage moves. That is the whole design, and it is exactly what a tier with opaque object identity cannot express directly.

## Purpose

The renderer sorts every visible draw into buckets each frame — by material pass, by depth — and the number of buckets is a few hundred, different every frame, and known only as the frame is walked. A node-based map allocates per bucket and frees them all at frame end; that is thousands of allocator round trips per frame for data whose lifetime is exactly one frame.

This container answers that with one decision: **storage is never released, only reused.** Emptying it sets the occupancy back to zero and keeps the array. After a few frames the array is as large as the worst frame needs, and from then on a frame's worth of bucketing costs zero allocations. The name is a little misleading — nothing about it is fixed-size — and the useful name would be *arena-backed map*.

It is worth being explicit that this container is a poor general-purpose map: the tree is never rebalanced, so sorted insertion degenerates to a list. It is correct for its callers because their keys — pass pointers, depths — arrive in scene order, which is effectively arbitrary.

## State

```text
RECORD Node<K, V>
  key   : K
  value : V
  left  : optional<Node>     # an address INSIDE the owning array
  right : optional<Node>

RECORD FixedMap<K, V>
  nodes : list<Node>         # the arena
  used  : int                # how many are live; the root is always nodes[0]
  cap   : int                # how many exist
```

**Invariants** — several, and all of them are enforced by scattered code:

- Nodes `[0, used)` are live and form one tree rooted at index 0. Nodes `[used, cap)` are constructed but unreachable.
- Insertion order is arena order: node *i* was inserted before node *i+1*. Nothing depends on this except the "iterate in any order" traversal, which is really "iterate in insertion order".
- **Every link is an address into `nodes`, so growing the arena invalidates every link.** The grow path repairs them by converting each link to an index against the old base and back against the new one. Getting this wrong produces a tree that points into freed memory and works for a while.
- Any outstanding reference to a node is invalidated by any insert that grows the arena. The insertion walk itself is written to survive this — see below.
- The container **cannot be moved**, only copied; the source says so in its own comment and the reason is the one above: a move would leave links pointing at the source's storage. A rebuild expressing links as indices removes the restriction entirely, and should.

## Growth

```text
FUNCTION grow()
  IF the multiplier is greater than one
    new_cap := 64 on the first allocation, ELSE cap * multiplier
  ELSE
    new_cap := cap + 64                # additive growth, for a fixed multiplier of 1
  allocate new_cap nodes
  copy the live ones over
  FOR EACH live node
    rewrite each of its two links from an offset against the old base
      to the same offset against the new base
  release the old storage
```

**Notes** — the growth policy is a template parameter, defaulting to doubling, with the additive path kept for callers that want a bounded overshoot. Sixty-four is the first-allocation size and the additive step; it is not derived from anything.

The copy is a raw block move for plain-data values and an element-wise copy otherwise. That distinction is incidental — what survives is that copying must preserve the *relative* positions of live nodes, because the link repair depends on offsets being stable.

## `insert`

**Contract** — descend from the root comparing keys; on an exact match return the existing node and change nothing; otherwise append a node to the arena and hang it off the leaf the walk stopped at. Returns the node. Grows the arena when it is full, invalidating every outstanding node reference. Never fails except by exhausting memory, which is fatal.

```text
FUNCTION insert(key) -> Node
  IF nothing is live THEN RETURN append(key)          # becomes the root
  node := root
  LOOP
    IF key ORDERED BEFORE node.key
      IF node.left EXISTS THEN node := node.left; CONTINUE
      child := append_under(node, key); node.left := child; RETURN child
    ELSE IF node.key ORDERED BEFORE key
      IF node.right EXISTS THEN node := node.right; CONTINUE
      child := append_under(node, key); node.right := child; RETURN child
    ELSE
      RETURN node                                     # exact match
```

**Invariants** — `append_under` is the subtle part and is the reason it exists as a separate step: appending may grow the arena, which moves the parent. So it records the parent's *index* first, appends, and then recomputes the parent's address from the index before the caller writes the link. A rebuild using indices throughout deletes this dance.

The two-argument form sets the value after locating the node, so it overwrites on a repeat key.

## `insert_anyway`

**Contract** — the same walk, but a key equal to a node's goes **left** instead of matching, so equal keys produce separate nodes. The result is a multi-map that still supports ordered traversal, at the cost of never being able to find "the" entry for a key.

**Notes** — the renderer uses the keyed form for material buckets, where two draws with the same pass must share a bucket, and the equal-keys form where the key is a sort depth and two objects at the same depth are genuinely two entries.

## `clear`

**Contract** — set the live count to zero. **Does not run any per-element teardown and does not release storage.** Constant time regardless of size.

**Invariants** — this is the container's reason to exist, and it is also its sharpest edge: a value type that owns a resource leaks on every clear. Every instantiation in the engine holds plain data or a pointer to something owned elsewhere. A rebuild must either keep that restriction and state it, or run teardown and lose the constant-time reset.

## `destroy`

**Contract** — tear down every node up to the *capacity*, not the live count, and release the storage. Resets to empty. This is the only path that returns memory.

**Notes** — tearing down to capacity rather than to occupancy is correct here, because the grow path constructs the whole array, not just the used prefix. It pairs with `setup` below.

## Traversal

**Contract** — five orders, all non-allocating except where noted:

- **left-to-right** — in-order, so ascending by key. Recursive.
- **right-to-left** — reverse in-order, descending by key. Recursive.
- **any order** — a linear sweep of the live prefix, which is insertion order. This is the cheap one and the one to reach for when order does not matter, because it touches memory linearly instead of chasing links.
- the same three, collecting into a caller-supplied sequence — either the values or pointers to the nodes.

**Notes** — the traversals take a raw function rather than a closure, so a caller needing context passes an identifier alongside. That is a 2007 constraint and disappears in a rebuild.

The recursion depth is the tree's height, which on degenerate input is the element count. With a few hundred elements and arbitrary keys that is fine; it is a real limit on using this container for anything larger or for sorted input.

## `setup`

**Contract** — invoke a callback on **every** node in the arena, live or not, from index zero to capacity. Used once after construction to give every node a value that is expensive to build and cheap to reuse — the renderer pre-sizes its bucket lists this way, so that a bucket reached for the first time already has its storage.

**Invariants** — this is the escape hatch that makes the never-free policy work for values that are themselves containers: the node's value survives `clear`, so a bucket's list keeps its capacity across frames and the whole structure converges to zero allocations. It also means the `clear` restriction above is not quite "plain data only" but the more precise "values whose stale contents are harmless and whose storage is worth keeping".

## Accessors

**Contract** — `size`, `empty`, `allocated` (capacity), `allocated_memory` (capacity in bytes), indexing by arena position, `begin`/`end` over the live prefix, and `first`/`last` over the whole arena — the latter pair explicitly for the setup path, not for iteration. `at(key)` and indexing by key both insert-if-absent and return the value, exactly as the insert path does.

**Notes** — the presence of two nearly identical pairs of bounds (`begin`/`end` over live, `first`/`last` over capacity) with no naming clue which is which is a genuine hazard, and the one place where reading this container's call sites is unavoidable. A rebuild should name them `live` and `storage`.
