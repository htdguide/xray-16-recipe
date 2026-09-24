# src/Layers/xrRender/Shader.h

> Declares the compiled material tree — shader, element, pass, and the bound lists — implemented in [`Shader.cpp`](Shader.cpp.md).

**Needs** — [`SH_Atomic.h`](SH_Atomic.h.md) · [`SH_Texture.h`](SH_Texture.h.md) · [`SH_Matrix.h`](SH_Matrix.h.md) · [`SH_Constant.h`](SH_Constant.h.md) · [`SH_RT.h`](SH_RT.h.md) · [`r_constants.h`](r_constants.h.md)
**Used by** — [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`DetailModel.cpp`](DetailModel.cpp.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`IRenderDetailModel.h`](IRenderDetailModel.h.md) · [`ParticleEffectDef.cpp`](ParticleEffectDef.cpp.md) · [`ParticleEffectDef.h`](ParticleEffectDef.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md) · [`ResourceManager.cpp`](ResourceManager.cpp.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) · [`ResourceManager_Scripting.cpp`](ResourceManager_Scripting.cpp.md) · [`Shader.cpp`](Shader.cpp.md) · _and 29 more_
**Tier floor** — T1: the record layouts are declared with explicit packing because they are compared and hashed as units.

## Purpose

Declares the records every other file in the chapter passes around, and the reference-counted handles to them. Their fields, the six-slot render-mode convention, the sort-key flags and every invariant are in [`Shader.cpp`](Shader.cpp.md).

## Exported units

- `Shader` — a material: six optional render-mode variants.
- `ShaderElement` — one variant: draw-order flags plus at most two passes.
- `SPass` — one pass: state block, program slots, merged constant table, and the texture/constant/matrix lists.
- `STextureList` — the (stage, texture) bindings, with `equal`, `clear`, `find_texture_stage` and `create_texture`.
- `SConstantList` · `SMatrixList` — up to four animated colour constants and texture-coordinate matrices.
- `SGeometry` — a vertex declaration plus vertex and index buffers, with its cached stride.
- `ref_shader` · `ref_selement` · `ref_pass` · `ref_geom` and friends — the counted handles, each of which creates through the resource registry and releases back into it.
- `SE_R1_*` — the names of the six render-mode slots for the forward generation.
- The pass cap, `SHADER_PASSES_MAX` = 2.
