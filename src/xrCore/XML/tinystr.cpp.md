# src/xrCore/XML/tinystr.cpp

> The three operations that allocate, and the growth rule that keeps a parser's repeated appends from being quadratic.

**Needs** — [`tinystr.h`](tinystr.h.md) · [`xrMemory.h`](../xrMemory.h.md) · [Seam: Allocator](../../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`tinystr.h`](tinystr.h.md)
**Tier floor** — T1: it allocates a header and its characters as one block and does its own pointer arithmetic to reach them.

## Purpose

Holds the out-of-line half of [`tinystr.h`](tinystr.h.md): reserving capacity, replacing contents, and extending them. These three are the only operations that touch the allocator, and the one decision in the file is how much to ask for.

## State

The shared empty representation — zero size, zero capacity, a single terminator — is a process-wide constant that every default-constructed string points at. It is never freed, and every release path must recognize it.

## `reserve`

**Contract** — ensures capacity for at least the requested character count. Shrinking is a no-op: an existing capacity is never given back. Allocates a new block, copies the live characters, releases the old one unless it is the shared empty representation.

## `assign`

**Contract** — replaces the contents with a given byte range. If the existing capacity already suffices, the bytes are copied in place and the size updated — no allocation. Otherwise a block of exactly the requested length is allocated and the old one released.

**Notes** — the reuse path is what makes re-assigning a node's value in a loop cheap, and the exact-fit allocation on the growing path is why `assign` does *not* over-allocate the way `append` does: an assignment is a complete replacement and the final size is known.

## `append`

**Contract** — extends the contents with a given byte range, growing capacity if needed.

```text
FUNCTION append(bytes)
  new_size <- size + length(bytes)
  IF new_size > capacity
    reserve(new_size + capacity)        # the growth rule
  copy bytes to the end; size <- new_size; terminate
```

**Invariants** — the growth rule asks for the needed size *plus the current capacity*, which is at least a doubling once the string is non-empty. That is what makes a parser that appends one character at a time — which is exactly what the text and comment readers do — linear rather than quadratic. A rebuild that reserves only the needed size reintroduces the quadratic behaviour and will be visibly slower on the larger shipped layout files.

**Notes** — the first append to an empty string reserves exactly the needed size, because the capacity being added is zero. Subsequent appends double. That is fine; the pathological case is the repeated one.

## Concatenation

**Contract** — the free concatenation operators reserve the sum of both lengths up front and then append, so building a string from two pieces costs one allocation.

**Notes** — the allocator underneath is the engine's own rather than the language's default, which is the one adaptation made to this vendored file and the only reason it appears in the recipe's dependency graph at all. See [Seam: Allocator](../../../SYSTEM-REQUIREMENTS.md#seam-allocator).
