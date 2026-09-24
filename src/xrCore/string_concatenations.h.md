# src/xrCore/string_concatenations.h

> Joining strings without allocating: a bounded concatenation into a caller buffer, and a stack-allocating form used on the hot paths that build paths and names every frame.

**Needs** — [`string_concatenations_inline.h`](string_concatenations_inline.h.md) · [`string_concatenations.cpp`](string_concatenations.cpp.md) · [`xrDebug_macros.h`](xrDebug_macros.h.md) · [`xrMemory.h`](xrMemory.h.md)
**Used by** — [`string_concatenations.cpp`](string_concatenations.cpp.md) · [`string_concatenations_inline.h`](string_concatenations_inline.h.md) · [`xrCore.cpp`](xrCore.cpp.md) · [`xrCore.h`](xrCore.h.md) · [`xr_ini.cpp`](xr_ini.cpp.md)
**Tier floor** — T1: the stack-allocating form places the result on the caller's own stack frame and must probe for stack exhaustion before doing so.

## Purpose

The engine builds strings constantly and in places where allocation is unacceptable: resolving a virtual path on every file open, assembling a bone or texture name during a level load, formatting a log line inside the failure path. This file supplies two ways to join strings that never touch the heap, and one guard that keeps the second from taking the process down.

## `strconcat` — into a caller's buffer

**Contract** — Joins any number of strings into a caller-supplied buffer of known size and terminates the result. Returns the buffer. **The first source may alias the destination** — the common "append to what is already there" case — but no other overlap is defined. On any failure the buffer is emptied and returned, so the result is always a valid string.

Three failures are recognized, and what each does is the contract:

| Condition | Result |
|---|---|
| no destination, or zero size | returns nothing |
| destination size above 256 MB | empties the buffer; the size is assumed to be a corrupted value rather than a real request |
| any source missing, or the sources exceed the buffer | empties the buffer |

```text
FUNCTION strconcat(destination, size, sources...) -> text
  # Copies byte by byte with a bound check per byte rather than measuring
  # first. Measuring would walk every source twice; the sources here are
  # short and the bound check is a register compare.
  cursor = destination
  limit  = destination + size - 1
  FOR EACH source IN sources
    IF source is none THEN empty the destination; RETURN
    WHILE source has a byte
      IF cursor == limit THEN empty the destination; RETURN
      copy the byte; advance both
  terminate at cursor
```

**Invariants** — Truncation is never silent: an overrun empties the result. A caller that gets back an empty string knows the join failed, and every caller that cares tests it.

**Notes** — The 256 MB ceiling on the *destination size* is a corruption detector, not a limit on strings: a buffer that large is a sign that a size argument was garbage, and the source says so.

## `STRCONCAT` — onto the caller's stack

**Contract** — Joins up to six strings into space allocated on the **caller's own stack frame**, and points a caller-supplied pointer at it. The result lives until the calling function returns. Nothing is freed and nothing is allocated on the heap.

```text
FUNCTION stack_concat(out_pointer, sources...) -> void
  # Three phases, and the order is forced: the size must be known before
  # the stack space is taken, and the space must be taken in the CALLER's
  # frame, which is why this cannot be an ordinary function.
  descriptor = measure each source once, keeping (pointer, length) pairs
  size = sum of lengths + 1
  IF size > 512 KB THEN report which source overran and fail fatally
  probe the stack for `size` bytes                  # see below
  out_pointer = stack space of `size` bytes in the caller's frame
  copy each source in order; terminate
```

**Invariants**

- **At most six sources.** Checked at build time.
- **At most 512 KB of result.** Exceeding it is fatal and the report names the source at which the running total overran, together with the first kilobyte of each source — because the usual cause is a string that was never terminated, and the report has to be readable.
- **The result's lifetime is the caller's frame.** Returning it, or storing it beyond the call, is a dangling pointer. This is the cost of the construct and the reason it is spelled in capitals at every use.

## Stack-overflow probe

**Contract** — Before taking a large amount of stack, the guard attempts the allocation inside a fault handler; if the stack cannot grow, it resets the guard page and returns, so the process survives with a failed join instead of dying. A comment in the source records the measurement that justifies keeping it: the probe costs about one percent of the construct's time.

**Notes** — This is the one place in the engine that recovers from stack exhaustion, and it only works on the platform whose runtime exposes a guard-page reset. Elsewhere the probe is a no-op and the construct will simply crash on an oversized join — which the 512 KB ceiling is there to prevent. A rebuild on a tier that grows stacks, or that has no stack-allocated strings, deletes this whole mechanism and keeps only the ceiling.
