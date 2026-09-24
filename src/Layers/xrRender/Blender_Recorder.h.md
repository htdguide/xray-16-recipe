# src/Layers/xrRender/Blender_Recorder.h

> Declares the material compiler's recording surface — the vocabulary a blender writes its pass list in.

**Needs** — [`tss.h`](tss.h.md) · [`Shader.h`](Shader.h.md) · [`r_constants.h`](r_constants.h.md)
**Used by** — [`Blender.cpp`](Blender.cpp.md) · [`Blender.h`](Blender.h.md) · [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) · [`Blender_Recorder_R2.cpp`](Blender_Recorder_R2.cpp.md) · [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md) · [`ResourceManager.cpp`](ResourceManager.cpp.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`ResourceManager_Scripting.cpp`](ResourceManager_Scripting.cpp.md) · [`TextureDescrManager.cpp`](TextureDescrManager.cpp.md) · [`BlenderDefault.cpp`](blenders/BlenderDefault.cpp.md) · [`BlenderDefault.h`](blenders/BlenderDefault.h.md) · [`Blender_Blur.cpp`](blenders/Blender_Blur.cpp.md) · [`Blender_Blur.h`](blenders/Blender_Blur.h.md) · [`Blender_BmmD.cpp`](blenders/Blender_BmmD.cpp.md) · _and 73 more_
**Tier floor** — T1: the method signatures carry graphics-API state tokens by value.

## Purpose

Declares the surface implemented across [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) (the shared front half and the fixed-function recorder), [`Blender_Recorder_R2.cpp`](Blender_Recorder_R2.cpp.md) (the programmable recorder) and [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md) (the constant binding). The substance is there; this file is the index.

It is worth reading once as a list, because **the set of methods here is the whole language a material template may speak**. A blender cannot reach the device, cannot allocate a texture, cannot decide a sort key: it can only call these.

## Exported units

**`CBlender_Compile`** — the recorder. Its members, by group:

- **Instance inputs** — the material instance's positional texture, constant and matrix name lists; the template being compiled; the element being filled; whether detail texturing was requested.
- **Compile outputs** — the resolved detail texture and its scaler, the detail diffuse/bump split, the steep-parallax decision, and (on the tessellating backend) which tessellation method the pass wants.
- **`SetParams`** — the two sort knobs.
- **Fixed-function recording** — `PassBegin`/`PassEnd`, the depth setter, the blend setters and their five named shorthands (blend, set, add, multiply, multiply-2x), the light/fog setter, the program setter; then `StageBegin`/`StageEnd` and the per-stage colour, alpha, address, transform, texture, matrix and constant setters, plus the canonical lightmap-stage template.
- **Programmable recording** — `r_Pass` (with and without a geometry program, plus tessellating and compute variants), the indexed sampler primitives (`i_Texture`, `i_Address`, `i_Filter*`, `i_Projective`, `i_BorderColor`, comparison filtering on the OpenGL path), the by-name sampler binders (`r_Sampler` and its clamped/point/linear shorthands), `r_Constant` to attach a per-frame binder to a named constant, `r_Stencil`/`r_StencilRef`/`r_CullMode`/`r_ColorWriteEnable`, and `r_End`.
- **`SampledImage`** — the name-driven sampler-plus-texture binder both recorders share.
- **`SetMapping`** — installs the standard per-frame constant binders; called by both recorders at the end of every pass.
- **`_cpp_Compile`** / **`_lua_Compile`** — the two entry points. The first compiles a template written as engine code; the second compiles one written as a script, which is how the material system stays extensible without a rebuild.

**Notes** — The invalid-sampler sentinel is the maximum integer value, and it is returned so often that it is part of the contract, not an error path: a blender emits bindings for every renderer generation it might run under and lets the compiled shader decide which exist.
