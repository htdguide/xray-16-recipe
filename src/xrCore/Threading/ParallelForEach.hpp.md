# src/xrCore/Threading/ParallelForEach.hpp

> The element-wise shorthand: the same range splitting, with the leaf loop written for you.

**Needs** — [`ParallelFor.hpp`](ParallelFor.hpp.md) · [`TaskManager.hpp`](TaskManager.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

[`ParallelFor.hpp`](ParallelFor.hpp.md) hands a body a whole sub-range and expects it to iterate. Most callers want per-element work. This file supplies the missing loop, and nothing else: it is a one-line adapter that exists so that the per-element form is the *default* at call sites and the range form is the explicit choice.

The split between the two files is arbitrary in the brief's sense — a rebuild should merge them, offering one operation with two body shapes.

## `xr_parallel_for_each`

**Contract** — runs a body once per element of a sequence, in parallel. The same four forms as the range version: blocking or not, parented or not. The grain is the default one — roughly one leaf per worker — and there is no way to override it through this entry point, which is the one real limitation of the shorthand.

```text
FUNCTION parallel_for_each(sequence, body, parent, wait) -> Task
  RETURN parallel_for(
    Range(start of sequence, end of sequence),
    LAMBDA sub_range: FOR EACH element IN sub_range: body(element),
    parent, wait)
```

**Notes** — the sequence is taken by mutable reference and the elements are handed to the body by mutable reference, so this is an in-place transform as much as a traversal. The body is responsible for the only thing that matters here: **two elements may be visited concurrently**, so a body that touches anything shared must synchronize it, and a body that resizes the sequence invalidates every other worker's cursor. Neither is checked.

The wrapping closure captures the caller's body by reference, so the body must outlive the parallel region. With the blocking form it always does; with the non-blocking form the caller owns that.
