# src/xrCore/dump_string.h

> Declares the debug renderings implemented in [`dump_string.cpp`](dump_string.cpp.md); absent from shipping builds.

**Needs** — [`dump_string.cpp`](dump_string.cpp.md)
**Used by** — [`dump_string.cpp`](dump_string.cpp.md) · [`vector.h`](vector.h.md)
**Tier floor** — T3: declarations.

## Purpose

Declares the math-type renderings described in [`dump_string.cpp`](dump_string.cpp.md). The whole file, declarations included, exists only in development builds — a call site that uses it must itself be conditional, which is the one thing a reader needs to know.

## Exported units

- **Render as text** — bool, 3-vector, 4x4 matrix, axis-aligned box.
- **Render with a name** — 3-vector and 4x4 matrix.
- **Render and log** — 3-vector and 4x4 matrix.
