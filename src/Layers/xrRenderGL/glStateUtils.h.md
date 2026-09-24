# src/Layers/xrRenderGL/glStateUtils.h

> Declares the eight lookups that map the engine's Direct3D 9 state vocabulary onto this API's enumerants.

**Needs** — [`glStateUtils.cpp`](glStateUtils.cpp.md)
**Used by** — [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) · [`glState.cpp`](glState.cpp.md) · [`glStateUtils.cpp`](glStateUtils.cpp.md)
**Tier floor** — T1: pure enumerant translation for a device call.

## Purpose

Declares the converters implemented in [`glStateUtils.cpp`](glStateUtils.cpp.md). They exist for exactly those state values whose engine-side numbering cannot be redefined to equal the device's — everything else is renamed for free in [`CommonTypes.h`](CommonTypes.h.md).

Exported units: `convert_fill_mode`, `convert_cull_mode`, `convert_comparison_func`, `convert_stencil_op`, `convert_blend_factor`, `convert_blend_op`, `convert_address_mode`, and `convert_texture_filter` — the last taking the current combined filter and a mip flag, because on this API minification and mip filtering share one enumerant.
