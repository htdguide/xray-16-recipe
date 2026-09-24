# src/xrCore/Memory/xrMemory_align.cpp

> Builds aligned allocation on top of an allocator that guarantees no alignment, by over-allocating and stashing the real address just below the aligned one.

**Needs** — [`xrMemory_align.h`](xrMemory_align.h.md) · [Seam: Allocator](../../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`xrMemory_align.h`](xrMemory_align.h.md)
**Tier floor** — T1: the whole file is address arithmetic on a raw heap block. There is no more device-facing code in the chapter.

## Purpose

Several things the engine hands to a foreign boundary need an address with a guaranteed alignment: four-wide float operands want sixteen bytes, and a few on-disk images are mapped onto structures with the same requirement. The underlying heap promises only enough alignment for the language's fundamental types. This file closes that gap the classic way — ask for more than you need, step forward to the first address that satisfies the constraint, and write the original address in the word immediately before the one you hand back.

It is entirely incidental in the brief's sense: a rebuild whose allocator already takes an alignment deletes this file. What survives is the requirement in [Seam: Allocator](../../../SYSTEM-REQUIREMENTS.md#seam-allocator) — sized allocation with sixteen-byte alignment, a resize, and a per-allocation size query — and the one structural idea below, which a rebuilder will meet again if they ever have to implement it themselves.

## The block layout

```text
| raw ... | back-pointer : address | gap | offset bytes | usable data ... | slack |
                                  ^                    ^
                                  |                    returned address + offset,
                                  |                    a multiple of alignment
                                  returned address minus gap
```

**Invariant** — the word immediately preceding `returned - gap` holds the address the underlying heap actually returned. Release and size-query recover it by rounding the given address *down* to a word boundary, stepping back one word, and reading. Nothing else may be stored there, and the caller must never write behind the address it was given.

**Invariant** — `gap` is the number of bytes needed to bring `offset` up to a word boundary, so that the back-pointer slot is itself word-aligned no matter how odd the requested offset is. When the offset is zero (the ordinary case) the gap is zero and the back-pointer sits directly under the returned address.

**Invariant** — the effective alignment is at least one machine word. A caller asking for less gets a word; a caller asking for a non-power-of-two is rejected.

## `aligned_offset_malloc` — and `aligned_malloc`, which is it with a zero offset

**Contract** — returns a block of at least the requested size, such that the address `offset` bytes into it is a multiple of `alignment`. Returns nothing on a non-power-of-two alignment or on heap exhaustion. If the offset would fall outside the requested size, the size is quietly grown to admit it — the offset is what the caller intends to align, so a block that cannot contain it is nonsense.

```text
FUNCTION aligned_offset_alloc(size, alignment, offset) -> optional<address>
  IF alignment is not a power of two
    FAIL WITH InvalidAlignment
  IF offset >= size AND offset != 0
    size <- offset + 1                       # make room for the aligned point
  align_mask <- max(alignment, word_size) - 1
  gap <- (-offset) AND (word_size - 1)       # bytes needed to word-align the slot

  raw <- heap_allocate(word_size + gap + align_mask + size)
  IF raw is none
    RETURN none

  # Step past the reserved word and the gap, round up to the alignment, then
  # step back by offset so it is the OFFSET point that lands on the boundary.
  result <- ((raw + word_size + gap + align_mask + offset) rounded down to alignment) - offset
  store raw in the word at (result - gap - word_size)
  RETURN result
```

**Notes** — the over-allocation is `word_size + gap + (alignment - 1)` bytes of overhead in the worst case, paid on every aligned block. That is the price of the scheme and the reason the engine does not route ordinary allocations through it.

## `aligned_offset_realloc` — and `aligned_realloc`

**Contract** — resizes a block previously obtained from one of the aligned allocators, preserving its contents up to the smaller of the old usable size and the new requested size, and preserving the alignment and offset relationship. A null block reallocates as a fresh allocation. A zero size releases and returns nothing. An offset outside the new size, or a non-power-of-two alignment, is an error. May return the same address, or a new one, exactly as any resize may.

```text
FUNCTION aligned_offset_realloc(block, size, alignment, offset) -> optional<address>
  IF block is none            -> RETURN aligned_offset_alloc(size, alignment, offset)
  IF size == 0                -> release(block); RETURN none
  IF offset >= size AND offset != 0  -> FAIL WITH InvalidOffset
  IF alignment is not a power of two -> FAIL WITH InvalidAlignment

  raw   <- the back-pointer stored under block
  shift <- block - raw                               # where the data sat in the raw block
  movable <- min(usable_size(raw) - shift, size)     # bytes worth preserving
  needed  <- word_size + gap + align_mask + size

  IF the old raw block has no room below the data for the NEW header and gap
    # widening the alignment would overwrite the header; a fresh block is the
    # only safe move
    raw2 <- heap_allocate(needed); must_release_old <- true
  ELSE
    raw2 <- try to grow the raw block in place
    IF that failed
      raw2 <- heap_allocate(needed); must_release_old <- true

  IF the block did not move AND the old returned address still satisfies the
     new alignment and offset
    RETURN block                                     # nothing to do

  result <- recompute the aligned address inside raw2, as in the allocator
  move `movable` bytes from the old data position to result
  IF must_release_old
    heap_release(raw)
  store raw2 in the word at (result - gap - word_size)
  RETURN result
```

**Notes** — the preserved byte count is computed from the *usable* size of the underlying block, not from the size the caller originally asked for, because nothing records the latter. That is what the size query exists for, and it is why the underlying allocator must provide one — a rebuild whose heap cannot answer "how big is this block really" has to record the requested size in the header alongside the back-pointer.

The copy uses a move that tolerates overlap, because an in-place grow can leave the new aligned address inside the old data's range.

The in-place-grow attempt is an optimization with a real payoff — growing a large aligned buffer without copying — and a real hazard: it may return a *different* address, at which point the old back-pointer is stale and must be rewritten, which the code does at the end unconditionally.

## `aligned_free` and `aligned_msize`

**Contract** — release recovers the back-pointer and releases the underlying block; a null address is a no-op. The size query recovers the back-pointer and reports the underlying block's usable size, which is at least what was requested plus the overhead; a null address reports zero. Neither validates that the address actually came from this allocator — passing a foreign address corrupts the heap silently, which is the usual bargain for a scheme like this.

```text
FUNCTION recover_raw(block: address) -> address
  slot <- (block rounded down to a word boundary) - word_size
  RETURN the address stored at slot
```

**Notes** — rounding *down* is what makes recovery work for a non-zero offset, where the returned address may itself be unaligned. It is also why nothing may be handed to release except an address one of these four allocators returned: any other address rounds down to a word holding something else.
