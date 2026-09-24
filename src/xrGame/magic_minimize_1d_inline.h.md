# src/xrGame/magic_minimize_1d_inline.h

> Writable access to the scalar minimizer's three tuning values.

**Needs** — [`magic_minimize_1d.h`](magic_minimize_1d.h.md)
**Used by** — [`magic_minimize_1d.h`](magic_minimize_1d.h.md)
**Tier floor** — T3: field reads

## Purpose

Carries three accessors out of the declaration. A rebuild folds them in.

## State

`Stateless.`

## `MaxLevel` · `MaxBracket` · `UserData`

**Contract** — hand out mutable references to the recursion depth limit, the refinement
iteration limit and the opaque caller value. They are mutable because the minimizer is
reused: a caller that fits many candidates with the same objective adjusts the budget
between fits rather than rebuilding it.
