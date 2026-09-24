# src/Layers/xrRender/r__occlusion.h

> Declares the hardware occlusion query pool and the allocation pattern its callers must honour.

**Needs** — [`r__occlusion.cpp`](r__occlusion.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Light_Package.cpp`](Light_Package.cpp.md) · [`light_smapvis.cpp`](light_smapvis.cpp.md) · [`light_vis.cpp`](light_vis.cpp.md) · [`r__occlusion.cpp`](r__occlusion.cpp.md) · [`r__pixel_calculator.cpp`](r__pixel_calculator.cpp.md)
**Tier floor** — T2: a declaration, plus two sizing constants.

## Purpose

Declares the pool implemented in [`r__occlusion.cpp`](r__occlusion.cpp.md).

## Sizing

```text
base_count  = 768                       # queries provisioned per parallel context
initial     = 2 * base_count * parallel_context_count
```

Neither is a limit — the pool grows on demand — and the base count is a guess at a busy frame's query load with margin. It is also the *growth quantum*: when the pool grows, it reserves another base count rather than one entry, so growth is a handful of reallocations rather than hundreds.

## The contract on callers

The header states, as prose, the usage pattern the pool's efficiency depends on: allocations happen in rounds, all of a round's queries are freed before the next round's allocations, and within a round the order repeats. Under that pattern the pool settles on a working set the size of one round. A caller that holds a query across many frames, or allocates and frees in an interleaved pattern, does not break correctness but defeats the reuse and makes the pool grow without bound.

This is worth carrying into a rebuild verbatim, because it is a property of the callers that nothing in the pool enforces.

## Exported units

- **`QueryPool`** (`R_occlusion`) — `create`, `destroy`, `begin`, `end`, `get`. Described in [`r__occlusion.cpp`](r__occlusion.cpp.md).
- **The result type** — the fragment count's width differs between backends (a sixty-four-bit count on one, thirty-two on the other). Nothing in the engine cares about the difference: every caller compares against zero or a small threshold. A rebuild should pick one width and be done.
