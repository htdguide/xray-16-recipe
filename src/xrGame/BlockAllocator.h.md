# src/xrGame/BlockAllocator.h

> A bump allocator over reusable fixed-size blocks: appends objects cheaply, and resets to empty without freeing anything.

**Needs** — _(none beyond the allocator seam)_ · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it exists to control allocation; in a tier with a compacting collector it has no reason to exist

## Purpose

Some per-frame computations in this chapter build a large collection of small records,
consume them, and throw them away — and do it again the next frame. Allocating each record
individually costs more than the computation. This is the answer: a growing list of
fixed-size blocks that objects are appended into by bumping a cursor, and an *empty*
operation that resets the cursors without releasing any memory, so the second frame and
every frame after it allocate nothing at all.

It is a pure optimization and it is the most incidental file in the chapter. A rebuild on a
tier with a generational collector should delete it and use an ordinary growable list; a
rebuild on a manual tier should keep the *idea* — reset without free — because the reuse is
where the win is, not the block structure.

## State

```text
RECORD BlockAllocator<T, BLOCK_SIZE>
  blocks         : list<pointer to array of BLOCK_SIZE T>   # owned; never shrinks
  block_count    : int    # how many blocks are currently IN USE
                          # invariant: <= blocks.length
  current_block  : pointer to the block being filled, or none
  block_position : int    # the bump cursor within that block
                          # invariant: 0 <= position <= BLOCK_SIZE
```

**Invariant** — a full block is represented by the cursor being *equal* to the block size,
and the next append is what rolls over to a new block. Initial state therefore sets the
cursor to the block size with no current block, so that the first append allocates.

**Invariant** — the live objects are exactly: every element of the first `block_count - 1`
in-use blocks, plus the first `block_position` elements of the current block. That is the
iteration order and it is the only correct way to enumerate the contents.

## `add`

**Contract** — returns a pointer to the next unused slot, allocating a new block when the
current one is exhausted. The returned memory is **raw**: nothing is constructed in it, and
the caller must initialize it. Never fails; never returns the same slot twice before a
reset.

```text
FUNCTION add() -> reference to T
  IF cursor is at the end of the block THEN
    IF every allocated block is already in use THEN allocate one more
    current_block = the next block in the list
    block_count += 1
    cursor = 0
  cursor += 1
  RETURN slot (cursor - 1) of current_block
```

## `empty`

**Contract** — declares every object dead. Resets the in-use count and the cursor, keeps
every allocated block for reuse. **Nothing is destructed** — the contained type must be one
for which that is acceptable, which in practice means plain data.

**Invariants** — after this call the enumeration above yields nothing, and the next append
reuses the first block's first slot.

## `clear`

**Contract** — releases every block and returns to the initial state. The only path that
actually frees.

## `for_each`

**Contract** — applies a callable to each live object, in insertion order, following the
enumeration invariant above. Returns immediately when the allocator has never been used.

**Notes** — the private section declares an index operator, a back accessor and two
construct helpers that nothing calls; one of them has a return type that cannot compile if
instantiated. They are unused scaffolding and a rebuild should omit them.
