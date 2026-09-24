# src/xrGame/magic_minimize_nd.h

> Declares a derivative-free minimizer over several variables on a box-shaped domain, built from repeated line searches.

**Needs** — [`magic_minimize_1d.h`](magic_minimize_1d.h.md) · [`magic_minimize_nd_inline.h`](magic_minimize_nd_inline.h.md)
**Used by** — [`magic_minimize_nd_inline.h`](magic_minimize_nd_inline.h.md) · [`min_obb.cpp`](min_obb.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `MinimizeND`, parameterized by the number of variables. See
[`magic_minimize_nd_inline.h`](magic_minimize_nd_inline.h.md) for the substance.

Exported units:

- `MinimizeND` — the objective over a vector, an owned one-dimensional minimizer, the
  iteration budget, the current point, the saved point, the direction set and its conjugate
  slot, and a scratch vector for the line objective.
- construction from the objective and the three budgets.
- `GetMinimum` — search a box-shaped domain from a starting point.
- `MaxLevel` · `MaxBracket` · `UserData` — the tuning values, the first two forwarded to the
  owned line minimizer.

**Notes** — the dimension is a compile-time parameter purely so that every buffer is a
fixed-size member and the search allocates nothing. What a rebuild needs is a minimizer
that does not allocate per call; whether the dimension is known at compile time is
incidental. The direction set is stored as one flat block with a separate array of pointers
into it, which is what makes the rotation at the end of each pass a pointer shuffle rather
than a copy — that *is* load-bearing, and a rebuild should rotate indices rather than move
rows.
