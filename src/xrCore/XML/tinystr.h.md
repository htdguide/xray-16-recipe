# src/xrCore/XML/tinystr.h

> Declares the parser's private string: one pointer, with the length, capacity and bytes in a single allocation behind it.

**Needs** — [`tinystr.cpp`](tinystr.cpp.md) · [`xrMemory.h`](../xrMemory.h.md)
**Used by** — [`tinystr.cpp`](tinystr.cpp.md) · [`tinyxml.cpp`](tinyxml.cpp.md) · [`tinyxml.h`](tinyxml.h.md) · [`tinyxmlparser.cpp`](tinyxmlparser.cpp.md)
**Tier floor** — T1: the representation is a deliberate single-pointer layout with a header stored ahead of the characters.

## Purpose

Declares the surface implemented in [`tinystr.cpp`](tinystr.cpp.md). This exists because the vendored parser predates being able to rely on a standard string type, and it survives because the *layout* it chose is genuinely cheaper for the parser's usage pattern: a node's value is one pointer wide, and an empty string costs no allocation at all.

A rebuild deletes this file and uses its language's string. Nothing about the type is visible in the data or in the engine's own interfaces.

## State

```text
RECORD String
  rep : pointer to Representation      # never null; points at the shared empty
                                       # representation when the string is empty

RECORD Representation                  # one allocation: header immediately
  size     : int                       # followed by `capacity + 1` bytes
  capacity : int
  chars    : bytes[capacity + 1]       # always zero-terminated at `size`
```

**Invariant** — the representation pointer is never null; an empty string points at a shared, statically allocated, zero-capacity representation. That is what makes default construction free and is why release must check for the shared object before freeing.

**Invariant** — the characters are always terminated at `size`, so the string can be handed to anything expecting a terminated byte sequence without a copy. The termination is maintained by every mutation.

## Exported units

- **`String`** — constructible empty, from a terminated byte sequence, from a byte range, or by copy.
- **`String.c_str`, `data`, `length`, `size`, `empty`, `capacity`** — the representation's fields.
- **`String.at`, `operator[]`** — indexed access.
- **`String.assign`, `append`, `operator=`, `operator+=`** — replacement and extension; see [`tinystr.cpp`](tinystr.cpp.md) for the growth rule.
- **`String.reserve`, `clear`, `swap`** — capacity and content management.
- **`operator==`, `operator<`, `operator>`, `operator+`** — comparison by bytes and concatenation.
- **`npos`** — the not-found marker returned by searches.
