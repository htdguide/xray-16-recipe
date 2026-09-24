# src/xrGame/magic_minimize_1d.h

> Declares a derivative-free search for the minimum of a scalar function on an interval.

**Needs** — [`magic_minimize_1d.cpp`](magic_minimize_1d.cpp.md) · [`magic_minimize_1d_inline.h`](magic_minimize_1d_inline.h.md)
**Used by** — [`magic_minimize_1d.cpp`](magic_minimize_1d.cpp.md) · [`magic_minimize_1d_inline.h`](magic_minimize_1d_inline.h.md) · [`magic_minimize_nd.h`](magic_minimize_nd.h.md) · [`magic_minimize_nd_inline.h`](magic_minimize_nd_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `Minimize1D`. See [`magic_minimize_1d.cpp`](magic_minimize_1d.cpp.md).

Exported units:

- `Minimize1D` — the objective function, a recursion depth limit, a refinement iteration
  limit, and an opaque value passed back to the objective on every call.
- construction from those four.
- `GetMinimum` — search an interval from a starting point, reporting where the minimum is
  and what it is.
- `MaxLevel` · `MaxBracket` · `UserData` — the three limits, writable after construction.
