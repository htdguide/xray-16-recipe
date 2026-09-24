# src/Layers/xrRenderDX11/StateManager/dx11State.h

> Declares the resolved per-pass state block: three pipeline objects, six sampler arrays, two reference values.

**Needs** — [`dx11State.cpp`](dx11State.cpp.md) · [`dx11SamplerStateCache.h`](dx11SamplerStateCache.h.md)
**Used by** — [`dx11State.cpp`](dx11State.cpp.md)
**Tier floor** — T1: a record of non-owning device handles.

## Purpose

Declares the surface implemented in [`dx11State.cpp`](dx11State.cpp.md). This is the type the material system holds one of per pass, and the type chapter 18's shader record means when it says "the state".

## Exported units

- **the pass state** — `Create` from a recorded state block, `Apply` onto a command list, `Release`.
- **`UpdateStencilRef` / `UpdateAlphaRef`** — the recorded block calls these while it replays, because those two values are not part of any device object here.
- **the sampler handle array type** — a list of cache handles, one per slot, shared with the sampler cache.

**Notes** — The constructor and destructor are public only because the project's allocation helpers require it; the type is meant to be created through `Create` and destroyed through `Release`, which is how a rebuild should express it.
