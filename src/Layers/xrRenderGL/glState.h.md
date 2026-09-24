# src/Layers/xrRenderGL/glState.h

> Declares the interned state block: the depth-stencil, blend, cull and per-stage sampler settings a material pass carries as one unit.

**Needs** — [`glState.cpp`](glState.cpp.md) · [`xrRender/SH_Texture.h`](../xrRender/SH_Texture.h.md)
**Used by** — [`CommonTypes.h`](CommonTypes.h.md) · [`glState.cpp`](glState.cpp.md)
**Tier floor** — T1: it owns device sampler objects whose lifetime is explicit.

## Purpose

Declares the state-block object implemented in [`glState.cpp`](glState.cpp.md). The shared material compiler records a pass's pipeline state as a stream of (name, value) pairs in Direct3D 9 numbering, interns the result, and expects one object back that can be applied as a unit. This declares that object, together with the two records it accumulates into — a depth-stencil record and a blend record, both mirroring the D3D9 field set exactly, because the compiler's value stream is in those terms.

Exported units:

- `create` — allocate an empty block, pre-set to the defaults listed in [`glState.cpp`](glState.cpp.md).
- `apply` — install the whole block on the device.
- `release` — destroy the block and its sampler objects.
- `update_render_state(name, value)` — fold one pipeline-state pair into the block.
- `update_sampler_state(stage, name, value)` — fold one sampler pair into the block's sampler object for that stage.
