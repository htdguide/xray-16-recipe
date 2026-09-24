# src/xrCore/string_concatenations_inline.h

> The measuring half of the stack-allocating join: turns a heterogeneous argument list into (pointer, length) pairs, computes the total, and copies.

**Needs** — [`string_concatenations.h`](string_concatenations.h.md) · [`string_concatenations.cpp`](string_concatenations.cpp.md) · [`xrstring.h`](xrstring.h.md) · [`../xrCommon/xr_string.h`](../xrCommon/xr_string.h.md)
**Used by** — [`string_concatenations.cpp`](string_concatenations.cpp.md) · [`string_concatenations.h`](string_concatenations.h.md)
**Tier floor** — T2: it reduces several string representations to a common (pointer, length) pair.

## Purpose

The stack-allocating join described in [`string_concatenations.h`](string_concatenations.h.md) needs to know the result's size before it can take the space, and the arguments may be raw text, interned strings or owning strings mixed freely. This file is the reduction: whatever the argument is, take its bytes and its length without copying.

Its separation from the caller-facing header is a compilation concern; a rebuild merges them.

## State

```text
RECORD JoinDescriptor            # constructed on the caller's stack, lives for the call
  parts : list<(pointer, length)>   # exactly 6 slots, statically sized
  count : int                       # how many are in use
```

**Invariants** — The slot count is six and is checked at build time. It is a limit of the construct, not of the format; six was enough for the longest path assembly in the engine.

## The reduction

**Contract** — For each argument, produce its byte pointer and its length without walking it more than once and without copying:

| Argument kind | Length from | Bytes from |
|---|---|---|
| raw text | measured, or zero when absent | the pointer itself |
| interned string | its stored length — a field read, free | its characters |
| owning string | its stored length | its buffer |

**Notes** — The interned-string case is the reason the descriptor exists at all: interned strings carry their length, so joining names costs no measurement. Since most joins in the engine are over names, the whole construct is close to a memory copy.

An argument that is absent contributes zero length but is still required to be non-empty at copy time; a null raw pointer is caught by an assertion rather than treated as an empty string.

## `size`

**Contract** — Sum of the lengths, plus one for the terminator. Fails into the error report when the sum exceeds 512 KB.

## `concat`

**Contract** — Copies each part into a caller-supplied destination in order and terminates. The destination must already be at least `size()` bytes; nothing is checked here because the caller just took exactly that much.

## `error_process`

**Contract** — Called only when the size ceiling is exceeded. Walks the parts again accumulating the running total to find **which** part crossed the ceiling, then hands that index and all the parts to the fatal reporter.

**Notes** — Finding the offending index rather than just reporting the total is the useful part: the cause is almost always one unterminated string, and its index names the call site's argument.
