# src/Layers/xrRender/SH_Matrix.h

> Declares the animated texture-coordinate matrix, implemented in [`SH_Matrix.cpp`](SH_Matrix.cpp.md).

**Needs** — [`SH_Matrix.cpp`](SH_Matrix.cpp.md) · [`xrEngine/WaveForm.h`](../../xrEngine/WaveForm.h.md)
**Used by** — [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`ResourceManager_Loader.cpp`](ResourceManager_Loader.cpp.md) · [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) · [`SH_Matrix.cpp`](SH_Matrix.cpp.md) · [`Shader.cpp`](Shader.cpp.md) · [`Shader.h`](Shader.h.md)
**Tier floor** — T1: it declares the byte layout the material library stores.

## Purpose

Declares the matrix record; the five modes, the centre-of-texture convention and the frozen serialization are in [`SH_Matrix.cpp`](SH_Matrix.cpp.md).

## Exported units

- `CMatrix` — the record: the computed matrix, the mode, the modifier flags, and five waveforms.
- `Calculate` — recompute for the current frame, at most once.
- `Load` · `Save` — the frozen mode/flags/five-waveform image.
- `Similar` — value equality, used to share one record between identical bindings.
- `tc_trans` — the texture-coordinate translation matrix, used by the scroll and centring steps.
- `modeProgrammable` · `modeTCM` · `modeS_refl` · `modeC_refl` · `modeDetail` — the five modes.
- `tcmScale` · `tcmRotate` · `tcmScroll` — the modifier flags, meaningful only in the `tcm` mode.
- `ref_matrix` — the counted handle.
