# src/Layers/xrRender/SH_Texture.h

> Declares the texture record and its counted handle, implemented in [`SH_Texture.cpp`](SH_Texture.cpp.md).

**Needs** — [`SH_Texture.cpp`](SH_Texture.cpp.md) · [`R_Backend.h`](R_Backend.h.md)
**Used by** — [`ColorMapManager.cpp`](ColorMapManager.cpp.md) · [`ColorMapManager.h`](ColorMapManager.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_Runtime.cpp`](R_Backend_Runtime.cpp.md) · [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) · [`SH_RT.h`](SH_RT.h.md) · [`SH_Texture.cpp`](SH_Texture.cpp.md) · [`Shader.cpp`](Shader.cpp.md) · [`Shader.h`](Shader.h.md) · [`Texture.cpp`](Texture.cpp.md) · [`dx11SH_Texture.cpp`](../xrRenderDX11/dx11SH_Texture.cpp.md) · [`glState.cpp`](../xrRenderGL/glState.cpp.md) · _and 2 more_
**Tier floor** — T1: it fixes the per-stage texture-slot numbering that the backend and the shipped shaders both index by.

## Purpose

Declares the texture record. Its state, its lazy-load rule and the three animation paths are in [`SH_Texture.cpp`](SH_Texture.cpp.md).

The header does decide one thing on its own, and it is frozen: **how a texture stage number encodes both the shader stage and the slot within it.** The renderer passes a single integer where the device wants a (stage, slot) pair, and the encoding is a set of bases:

```text
pixel slots    start at 0      and there are 16
vertex slots   start next      and there are 4
geometry slots start next      and there are 16
hull, domain, compute slots follow, 16 each, on the backend that has them
```

On the newest backend the bases are spaced **256 apart** rather than packed, because that device allows up to 128 textures per stage and the spacing must exceed the maximum so the two halves of the number never collide. On the OpenGL backend, which does not distinguish stages at the binding point, the bases are packed tight. A rebuild is free to carry a pair instead — but the *counts* above are what the shipped material scripts and the constant tables assume.

## Exported units

- `CTexture` — the texture record: load, unload, per-stage bind, dimensions, array-slice selection, video transport.
- `ref_texture` — the counted handle; creating one interns the name in the resource registry, and it also exposes the texture's companion bump name.
- `MaxTextures` — the per-stage slot counts, and their combined total.
- `ResourceShaderType` — the per-stage base offsets described above.
