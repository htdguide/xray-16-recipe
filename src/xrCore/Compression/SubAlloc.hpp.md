# src/xrCore/Compression/SubAlloc.hpp

> The statistical coder's private memory pool: one large block, carved from both ends, with free lists in 38 size classes and no general-purpose allocator anywhere.

**Needs** — [`PPMdType.h`](PPMdType.h.md) · [`PPMd.h`](PPMd.h.md)
**Used by** — [`Model.cpp`](Model.cpp.md)
**Tier floor** — T1: the model stores raw pointers into this pool and compares them against pool landmarks to answer questions about its own structure. Addresses are data here.

## Purpose

The model in [`Model.cpp`](Model.cpp.md) allocates and frees millions of tiny records — context nodes and symbol-statistics arrays — in a tight loop. A general allocator is far too slow and, worse, gives no way to answer the question the model asks constantly: *is this pointer inside the tree area or inside the text area?* This pool answers that with one comparison, because it lays the two areas out at known ends of one block.

It is also, deliberately, an allocator that **runs out**. The model's restoration policy is triggered by exhaustion, so the pool's capacity is a parameter of the compressed format, not a resource detail.

## Layout

One contiguous block. Three cursors divide it, and every allocation moves one of them.

```text
  HeapStart                 UnitsStart      LoUnit        HiUnit = end
  |-------------------------|---------------|-------------|
  | text area (grows right) | units area    | free gap    | context area
  |                         | (grows right) |             | (grows LEFT)
        pText ->                   LoUnit ->     <- HiUnit
```

- **Text area** — a plain byte log of symbols seen, appended one byte per coded symbol when the model is behind. A "successor" pointer into this area means *the successor context has not been built yet; here is where its bytes start*. The model distinguishes a real context from a deferred one by testing the pointer against `UnitsStart`.
- **Units area** — symbol-statistics arrays, allocated in multiples of a unit and served from the free lists.
- **Context area** — fixed-size context nodes, taken one at a time off the top.

**Invariants** — the split is set once at initialization: the units-plus-context region is **seven eighths** of the pool, rounded down to a whole number of units, and the text area gets the remaining eighth. The model checks `pText >= UnitsStart` every time it appends a byte; when that becomes true, the pool is exhausted and restoration runs. So the one-eighth text budget is what bounds how far behind the model may fall before it restarts, and changing it changes where restarts happen — and therefore the output bytes.

```text
RECORD BlockNode                     # head of one size class's free list
  stamp : int          # how many blocks the list holds
  next  : optional<BlockNode>

RECORD MemoryBlock : BlockNode       # a free block in the units area
  units : int          # its size, in units
  # invariant: a free block's stamp is all-ones; that marker is how a neighbour
  # is recognized as free during coalescing and during text-area recovery
```

### Size classes

```text
UNIT_SIZE = 12 bytes
N1 = N2 = N3 = 4                       # 4 classes stepping by 1, then by 2, then by 3
N4 = (128 + 3 - N1 - 2*N2 - 3*N3) / 4  # = 26 classes stepping by 4
N_INDEXES = N1 + N2 + N3 + N4          # = 38 size classes, covering 1..128 units
```

Two lookup tables, built once at startup, make the mapping constant-time in both directions: index to unit count, and unit count (1..128) to the smallest index that holds it. The step pattern — fine near the bottom, coarse near the top — is fitted to the model's actual demand, which is overwhelmingly for very small arrays with a long thin tail.

A unit is 12 bytes because that is the size of one context node and of two symbol-statistics entries on the original's 32-bit layout. **On a 64-bit target the packed records are larger than 12 bytes**, so the unit-to-byte conversion — written as `8*n + 4*n`, deliberately not as a multiply — no longer describes the records it serves. This is the one place in the pool where the source's arithmetic and the source's structures can disagree, and a rebuild must derive the unit size from the record sizes rather than copying the constant.

## `StartSubAllocator` / `StopSubAllocator`

**Contract** — `StartSubAllocator` takes a size **in megabytes**, shifts it up by twenty, and allocates that many bytes; if the pool is already exactly that size it keeps it and reports success without disturbing the model. Otherwise it releases the old pool first. It reports failure rather than aborting. `StopSubAllocator` releases the pool and is safe on an empty one.

**Notes** — the "already the right size" shortcut is what lets the engine call initialize repeatedly without churning thirty-two megabytes. Note that it also leaves the *model* intact, which is only safe because the model is re-initialized separately.

## `GetUsedMemory`

**Contract** — pool size, minus the untouched gap between the two inward-growing cursors, minus the untouched head of the text area, minus everything sitting on the free lists. Allocates nothing, walks only the 38 list heads.

**Invariants** — this is the number the restoration policy compares against half the pool and against three quarters of it when deciding whether a prune was productive. It must therefore be exact, not an estimate.

## Allocation

```text
FUNCTION alloc_units(n) -> optional<pointer>
  index := class_for(n)
  IF the free list for index is non-empty
    RETURN pop from it
  IF LoUnit + bytes(index) <= HiUnit         # take from the gap
    advance LoUnit and RETURN the old LoUnit
  RETURN alloc_units_rare(index)

FUNCTION alloc_context() -> optional<pointer>
  IF the gap is non-empty
    HiUnit := HiUnit - UNIT_SIZE             # contexts grow downward
    RETURN HiUnit
  IF the one-unit free list is non-empty THEN RETURN pop from it
  RETURN alloc_units_rare(one-unit class)
```

The two areas grow toward each other out of one gap, and whichever exhausts it first forces the slow path. Contexts come off the top precisely so that a context pointer is always numerically above the units area, which is another fact the model tests.

### `alloc_units_rare` — the slow path

```text
FUNCTION alloc_units_rare(index) -> optional<pointer>
  IF the glue budget is exhausted
    glue_free_blocks()                       # coalesce everything
    IF the requested class is now non-empty THEN RETURN pop from it
  search upward through the larger size classes
  IF one is found
    block := pop from it
    split_block(block, from: that class, to: index)   # remainder back on the lists
    RETURN block
  # nothing larger exists: steal from the text area's tail
  spend one unit of glue budget
  IF the text area has room to give back bytes(index)
    UnitsStart := UnitsStart - bytes(index)
    RETURN UnitsStart
  RETURN none                                # the model must now restore itself
```

**Invariants** — `split_block` puts the remainder back as at most *two* blocks: if the leftover unit count does not land exactly on a size class, one block of the next class down is carved off first and the rest is filed at its exact class. This keeps every block on a list whose class it exactly matches, which is what makes coalescing able to trust the class tables.

Stealing from the text area is the pool's last resort and it **shrinks the model's memory of recent bytes**. It is also where the one-eighth split stops being a partition and becomes a starting point.

### `glue_free_blocks` — coalescing

```text
FUNCTION glue_free_blocks()
  place a zero stamp just above LoUnit so the walk terminates
  drain every size-class list onto one scratch list, and for each block
    WHILE the block immediately after it is marked free
      absorb it: add its unit count, and zero that neighbour's count
  FOR EACH surviving block
    chop off whole 128-unit pieces onto the largest class
    file the remainder at its exact class, carving one smaller block first
      if it does not land on a class boundary
  reset the glue budget to 8192
```

**Invariants** — adjacency is tested by *address arithmetic*: the block starting at `p + p.units` is the next one. That works only because the units area is never interleaved with anything else and every block records its true size. The free marker is the all-ones stamp, which no live record can hold because a live node's stamp field is a list count.

The glue budget — 8192 allocations between coalescing passes — is a throttle: coalescing walks every free list and is far too expensive to do on every miss. The number is not derived from anything and is one of the values in this file that reads as tuned by measurement rather than reasoned.

## Resizing in place

```text
FUNCTION expand_units(p, old_n) -> optional<pointer>
  IF old_n and old_n + 1 land in the same size class THEN RETURN p   # free growth
  q := alloc_units(old_n + 1)
  IF q EXISTS
    copy old_n units from p to q
    file p back on its class
  RETURN q

FUNCTION shrink_units(p, old_n, new_n) -> pointer
  IF both land in the same class THEN RETURN p
  IF the target class has a free block
    move into it, file p back, RETURN the new block
  split_block(p, old_n's class, new_n's class)       # keep the address
  RETURN p
```

**Notes** — the coarse size classes are what make growth usually free: a statistics array that gains one symbol almost always still fits its class. That is the reason the classes step by 2, 3 and 4 units rather than by 1 throughout.

`shrink_units` **prefers to move** when the smaller class has a block waiting, even though splitting in place would also work. That is deliberate compaction toward the low end of the pool, and it pairs with `move_units_up` below.

## Compaction helpers

```text
FUNCTION move_units_up(p, n) -> pointer
  # called during a prune, to pull live data down out of the text area's path
  IF p is more than 16 KiB above UnitsStart
     OR p already sits below the head of its free list
    RETURN p                                  # not worth moving
  q := pop from that class's list
  copy n units into q
  IF p was exactly at UnitsStart THEN give its bytes back to the text area
  ELSE file p back on its class
  RETURN q

FUNCTION free_units(p, n)
  file p at n's class

FUNCTION free_one_unit(p)
  IF p is exactly at UnitsStart
    mark it free and hand the unit back to the text area
  ELSE file it on the one-unit list

FUNCTION expand_text_area()
  WHILE the record sitting at UnitsStart is marked free
    absorb it: UnitsStart moves up past it, count it, and clear its marker
  unlink every counted block from the free list it is on
```

**Invariants** — `expand_text_area` is the only way the boundary between the text area and the units area moves back the other way, and it runs after a prune, when many blocks at the bottom of the units area have just been freed. It relies on the free marker and on the two-pass structure — count first, unlink second — because a block cannot be walked and unlinked in the same traversal.

The 16 KiB threshold in `move_units_up` bounds how much work compaction does: data already far from the boundary is left alone. The number is a tuning constant with no derivation in the source.

**Notes** — the whole file is a single global pool with no reentrancy, which is why the compressor is serialized process-wide. In a rebuild, the pool is an object; the model holds one; two compressions run at once. Nothing about the *format* requires the globals.

The pool never returns memory to the system between compressions — `StopSubAllocator` is called only at shutdown — so the thirty-two megabytes the engine asks for are resident for the life of the process. That is a real cost of the design and a rebuild may reasonably decide to allocate the pool per operation, at the price of touching that much memory each time.
