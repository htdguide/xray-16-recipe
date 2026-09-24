# src/Layers/xrRender/SH_Atomic.h

> Declares the atomic device resources — one program record per shader stage, the state block, the vertex declaration — defined in [`SH_Atomic.cpp`](SH_Atomic.cpp.md).

**Needs** — [`SH_Atomic.cpp`](SH_Atomic.cpp.md) · [`r_constants.h`](r_constants.h.md) · [`tss_def.h`](tss_def.h.md) · [`xrCore/xr_resource.h`](../../xrCore/xr_resource.h.md)
**Used by** — [`ResourceManager.h`](ResourceManager.h.md) · [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md) · [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) · [`SH_Atomic.cpp`](SH_Atomic.cpp.md) · [`Shader.cpp`](Shader.cpp.md) · [`Shader.h`](Shader.h.md) · [`ShaderResourceTraits.h`](ShaderResourceTraits.h.md)
**Tier floor** — T1: the records are explicitly packed because they are compared and hashed as byte units.

## Purpose

Declares the leaves of the compiled material tree and the counted handle for each. Their fields, their invariants and the self-unregistration discipline are in [`SH_Atomic.cpp`](SH_Atomic.cpp.md).

## Exported units

- `SVS` · `SPS` · `SGS` · `SHS` · `SDS` · `SCS` — the vertex, pixel, geometry, hull, domain and compute program records; each is named, holds a device handle, and carries the named-constant table reflected out of its compilation.
- `SPP` — OpenGL backend only: a program pipeline object binding several separable programs, or a linked monolithic program when the device lacks separable programs.
- `SInputSignature` — newest backend only: a compiled vertex program's input description, shared by every input layout built against it.
- `SState` — a device state object plus the recorded state assignments it was compiled from.
- `SDeclaration` — a vertex declaration: the backend-neutral element list plus the device layout(s) built from it.
- `ref_vs` · `ref_ps` · `ref_gs` · `ref_hs` · `ref_ds` · `ref_cs` · `ref_pp` · `ref_state` · `ref_declaration` · `ref_input_sign` — the counted handles.
