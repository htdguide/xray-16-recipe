# src/Layers/xrRenderDX11/StateManager/dx11StateManager.h

> Declares the per-command-list state front end and the three-flag reconciliation it runs per state kind.

**Needs** — [`dx11StateManager.cpp`](dx11StateManager.cpp.md) · [`dx11StateCache.h`](dx11StateCache.h.md)
**Used by** — [`dx11StateManager.cpp`](dx11StateManager.cpp.md) · [`dx11R_Backend_Runtime.h`](../dx11R_Backend_Runtime.h.md)
**Tier floor** — T1: holds non-owning device handles and driver description structures.

## Purpose

Declares the surface implemented in [`dx11StateManager.cpp`](dx11StateManager.cpp.md). One instance belongs to each command list; it holds no ownership, since every object it binds is owned by a process-wide cache.

## Exported units

- **fast path** — `SetRasterizerState`, `SetDepthStencilState`, `SetBlendState`, `SetStencilRef`, `SetAlphaRef`: take already-resolved objects and values. This is what a material pass uses.
- **slow path** — `SetStencil`, `SetDepthFunc`, `SetDepthEnable`, `SetColorWriteEnable`, `SetCullMode`, `SetFillMode`, `SetMultisample`, `SetSampleMask`, `EnableScissoring`: individual fields in the engine's legacy vocabulary. These may create a new state object; they are routed through the command list rather than called directly, so that a different backend can translate them its own way.
- **`OverrideScissoring`** — force the scissor bit across whatever passes set.
- **`BindAlphaRef` / `UnmapConstants`** — attach and detach the shader constant that carries the alpha cut-off.
- **`Apply`** — reconcile and bind; called once per draw.
- **`Reset`** — return to defaults.

**Notes** — The source marks the legacy enumerations as a confusion hazard: the field setters take plain integers carrying the *previous* graphics generation's enumeration values, translated on the way in. A rebuild should give the engine's own enumerations names and let the backend translate from those, which removes the hazard entirely.
