# src/Layers/xrRender/light_smapvis.h

> Declares the per-light, per-context shadow-caster visibility cache.

**Needs** — [`light_smapvis.cpp`](light_smapvis.cpp.md) · [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md)
**Used by** — [`light.cpp`](light.cpp.md) · [`light.h`](light.h.md) · [`light_smapvis.cpp`](light_smapvis.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the type implemented in [`light_smapvis.cpp`](light_smapvis.cpp.md). Each light owns one instance per render context; the state machine, the learning rule and the marking trick are described there.

Exported units:

- **`ShadowCasterVis`** (`smapvis`) — implements the graph builder's caster-feedback callback, so the graph can hand it the *n*-th caster it produces without the cache knowing how the graph is walked. It exposes `begin`, `end`, `mark`, `flush_query`, `reset_query`, `invalidate` and a test for whether the sleep interval has elapsed.
