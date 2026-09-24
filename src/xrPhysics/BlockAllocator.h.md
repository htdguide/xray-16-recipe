# src/xrPhysics/BlockAllocator.h

> A bump allocator whose whole content is reclaimed at once, for the per-step
> scratch data the solver hands back.

**Needs** — _(none beyond the allocator)_ · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`Physics.h`](Physics.h.md)
**Tier floor** — T1: it exists because the records it hands out are addressed by the
dynamics library after they are handed over, so they must not move and must not be
reference-counted.

## Purpose

Every physics step produces a few hundred short-lived records — one per contact that needs a
force reading, one per contact that needs a drag effect applied — which are written by the
solver during the step and read once afterwards. Allocating each individually is the
frame-budget problem named in the
[tier justification](../../SYSTEM-REQUIREMENTS.md#2-tier); a growable array is wrong because
the solver holds pointers into it across the step.

The answer is a chain of fixed-size blocks that is emptied, not freed, at the end of each
step: capacity ratchets up to the worst case seen and then stops allocating entirely.

## State

```text
RECORD BlockAllocator<T, block_size>
  blocks         : list<block of block_size T>   # owned; never shrinks
  block_count    : int    # how many blocks are in use this cycle
  block_position : int    # next free slot in the current block
  current_block  : block  # invariant: blocks[block_count - 1] when block_count > 0
  # invariant: 0 <= block_position <= block_size; equality means the block is full
  # invariant: a pointer handed out stays valid until empty() is called
```

## `add`

**Contract** — returns a pointer to the next free record, advancing into the next block when
the current one is full and allocating a new block only when the chain has run out. Never
fails, never moves previously handed-out records, never initializes the record it returns —
the caller writes every field.

## `empty`

**Contract** — marks every record free and rewinds to the first block, without releasing any
memory. This is the once-per-step call. Its cost is constant regardless of how many records
were handed out, which is the entire point.

## `clear`

**Contract** — releases every block. Called only when the world is torn down.

## `for_each`

**Contract** — visits every record handed out since the last `empty`, in allocation order,
and does nothing when none were. Used to apply the accumulated effects at the end of a step.

**Notes** — order of visitation is allocation order, and at least one user depends on it: the
drag effectors are merged per body as they are created, so a later record for the same body
must be seen after the earlier one.

The block size is a template parameter fixed at each use site; both current users pick 128
records, which is roughly the contact count of a busy ragdoll. Picking it too small costs a
block walk, too large costs resident memory — neither is load-bearing.
