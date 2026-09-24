# src/xrCore/xrPool.h

> A free-list allocator for one object type: fixed-size blocks, never returned to the general allocator until the pool dies, and the freed object's own storage holds the list link.

**Needs** — [`xrDebug_macros.h`](xrDebug_macros.h.md) · [`xrMemory.h`](xrMemory.h.md) · [`../xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`ISpatial.h`](../xrCDB/ISpatial.h.md)
**Tier floor** — T0-adjacent, T1: it reuses the storage of a *destroyed* object as a pointer field. That requires both that the object's memory outlive its lifetime and that the type be at least pointer-sized, neither of which most tiers let you say.

## Purpose

A few object types in the engine are created and destroyed in the thousands per second and always in the same size — path nodes, scheduler entries, particle records. For those, going through the general allocator on each one is the measurable cost. This pool trades memory for time: it allocates in blocks and never gives them back while it lives.

## State

```text
RECORD Pool of T with block size N
  free_head : optional<address>    # head of the free list
  blocks    : list<address>        # every block ever allocated, for teardown
```

**Invariants**

- **The free list is threaded through the free objects' own storage.** A slot that is not in use holds, in its first pointer-sized bytes, the address of the next free slot. Therefore `size(T) >= size(pointer)`; nothing checks it.
- **Nothing is ever freed to the general allocator except at teardown.** A pool's peak occupancy is its permanent footprint.
- Blocks are never coalesced and slots from different blocks interleave freely on the list; the list is not ordered.

## `create`

**Contract** — Returns a constructed object. Takes the head of the free list, or grows by one block when the list is empty. Never fails except by the general allocator failing.

```text
FUNCTION create() -> T
  IF free_head is none THEN grow_one_block()
  slot = free_head
  free_head = the pointer stored in slot     # the link, read before we construct over it
  RETURN construct T in place at slot
```

## `destroy`

**Contract** — Destroys the object and pushes its slot onto the free list, then nulls the caller's pointer.

```text
FUNCTION destroy(object) -> void
  run the destructor of object
  store free_head into the object's storage   # storage reused as the link
  free_head = address of object
  object = none                               # the caller's variable
```

**Invariants** — The destructor runs *before* the storage becomes a link. Reading the object after destroying it reads a pointer.

## `grow_one_block`

**Contract** — Allocates `N` contiguous slots, records the block for teardown, and threads all of them onto the free list.

```text
FUNCTION grow_one_block() -> void
  REQUIRE the free list is empty              # we only grow when exhausted
  block = allocate N slots
  remember block
  # Link slot i to slot i+1 for i in [0, N-1), then terminate the last.
  # The loop runs N-1 times and the final slot is terminated separately:
  # writing N links would run one slot past the block.
  FOR i FROM 0 TO N - 2
    store address of slot[i+1] into slot[i]
  store none into slot[N-1]
  free_head = address of slot[0]
```

**Notes** — The source comments the "minus one" explicitly, which is the right instinct: the off-by-one here writes past the block and the partitioning loop is the only place it can happen.

## `clear` and teardown

**Contract** — Both free every block through the general allocator and forget them. Clearing additionally resets the free list, so the pool is reusable. **Neither runs any destructor** — every object still outstanding is abandoned. A pool is cleared only when its users are known to be gone.

**Notes** — The block size is a compile-time parameter of each pool, chosen per use site. Nothing measures it; the number is a guess about how many of that object exist at once, and the cost of guessing low is one extra block allocation, while the cost of guessing high is permanent memory.
