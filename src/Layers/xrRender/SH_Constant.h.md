# src/Layers/xrRender/SH_Constant.h

> Declares the animated colour constant, implemented in [`SH_Constant.cpp`](SH_Constant.cpp.md).

**Needs** — [`SH_Constant.cpp`](SH_Constant.cpp.md) · [`xrEngine/WaveForm.h`](../../xrEngine/WaveForm.h.md)
**Used by** — [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`ResourceManager_Loader.cpp`](ResourceManager_Loader.cpp.md) · [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) · [`SH_Constant.cpp`](SH_Constant.cpp.md) · [`Shader.cpp`](Shader.cpp.md) · [`Shader.h`](Shader.h.md)
**Tier floor** — T1: it declares the byte layout the material library stores.

## Purpose

Declares the constant record; its state, its once-per-frame evaluation rule and its frozen serialization are in [`SH_Constant.cpp`](SH_Constant.cpp.md).

## Exported units

- `CConstant` — the record: four per-channel waveforms, the evaluated colour in float and packed form, the frame memo.
- `Calculate` — evaluate at most once per frame.
- `Load` · `Save` — the frozen four-waveform byte image.
- `Similar` — value equality, used to share one record between identical bindings.
- `set_float` · `set_dword` — drive the value directly, which is what "programmable" mode means.
- `modeProgrammable` · `modeWaveForm` — the two modes.
- `ref_constant_obsolette` — the counted handle. Its name records the author's own verdict: this animation path predates programmable shading and survives only because the shipped material library contains records in this form.
