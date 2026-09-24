# src/Layers/xrRenderDX11/StateManager/dx11StateCache.cpp

> Instantiates the three state caches and supplies the one operation that differs between them: asking the device to create the object.

**Needs** — [`dx11StateCache.h`](dx11StateCache.h.md) · [`../dx11HW.h`](../dx11HW.h.md)
**Used by** — [`dx11StateCache.h`](dx11StateCache.h.md)
**Tier floor** — T1: direct device object creation.

## Purpose

The interning algorithm is shared; only "create this kind of object" is not. This file supplies that step for rasterizer, depth-stencil and blend state, and defines the three process-wide caches.

## `create a state object`

**Contract** — asks the device to build the object from a normalized description. A failure is fatal: every description reaching this point has already been normalized and validated, so a rejection means a programming error rather than a resource shortage.

## `ClearStateArray`

**Contract** — releases every cached object and empties the cache. Only legal at device teardown; see [`dx11StateCache.h`](dx11StateCache.h.md).

**Notes** — Each cache pre-reserves room for ten entries. That number is a statement about the design, not a tuning constant: a whole game level resolves to on the order of ten distinct objects per kind, which is the evidence that interning is worth doing at all.

Debug builds log each creation with a running count, which is how the ten-ish figure is checked; the source marks the logging for removal.
