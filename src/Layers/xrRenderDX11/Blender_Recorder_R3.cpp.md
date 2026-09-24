# src/Layers/xrRenderDX11/Blender_Recorder_R3.cpp

> The backend half of the material compiler: what a pass declaration turns into here — six programs, a merged constant table, named samplers with fixed presets, and the stencil vocabulary.

**Needs** — [`xrRender/Blender_Recorder.h`](../xrRender/Blender_Recorder.h.md) · [`xrRender/Blender.h`](../xrRender/Blender.h.md) · [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md) · [`dx11r_constants.cpp`](dx11r_constants.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it is load-time material compilation; only the resources it produces are device-facing.

## Purpose

Chapter 18 owns the material compiler; this file supplies the parts that only make sense on a backend with separate programmable stages and separately declared samplers. The load-bearing content is what happens when a material script opens a pass.

## `r_Pass`

**Contract** — open a render pass: reset the recorded state, the constant table and the texture, matrix and constant lists; apply the depth, blend, alpha-test and fog settings the pass declared; create (or find) the vertex, geometry and pixel programs; and **merge all three programs' constant tables into one**. The tessellation and compute stages are bound to a null program, so that a pass which does not use them leaves nothing from a previous pass behind.

```text
FUNCTION open_pass(vertex_name, geometry_name, pixel_name, fog, depth..., blend..., alpha...)
  reset recorded state, constant table, texture/matrix/constant lists
  apply depth test and write, blend mode, alpha test, fog
  pixel_program = create_pixel_program(pixel_name)
  compile_flags = pixel_program declares the legacy dialect
                  ? enable backwards compatibility : none
  vertex_program   = create_vertex_program(vertex_name, compile_flags)
  geometry_program = create_geometry_program(geometry_name)
  tessellation and compute stages := the null program
  constant_table = merge(pixel, vertex, geometry)
  IF the pixel program is the null program THEN
    record the legacy "last stage disabled" marker
```

**Invariants** — **The merged table is what makes by-name constant binding work across stages.** A name declared by two stages becomes one entry with two per-stage placements, which is exactly what the fan-out in [`dx11r_constants_cache.h`](dx11r_constants_cache.h.md) consumes.

**The compile flags come from the pixel program and are applied to the vertex program.** The legacy shader dialect must be compiled with a compatibility switch, and whether a material is written in that dialect is discovered when the pixel program is compiled first. So the order here is not incidental: pixel, then vertex with the flags the pixel compile revealed.

## `r_TessPass` / `r_ComputePass`

**Contract** — open a pass that additionally uses hull and domain programs, or one that is a compute dispatch instead of a draw. Both build on the ordinary pass and merge the extra stages' constants into the same table.

## `r_dx11Sampler` — the named sampler presets

**Contract** — declare a sampler by name and give it a preset configuration. **The name is the configuration**: a small fixed set of names each imply an addressing mode and a filter, and an unrecognized name gets the defaults.

```text
smp_nofilter  clamp,  nearest, no mip filtering        # exact texel fetches
smp_rtlinear  clamp,  linear,  no mip filtering        # sampling a full-screen target
smp_linear    wrap,   linear with linear mips          # ordinary repeating texture
smp_base      wrap,   anisotropic                      # the surface's main texture
smp_material  clamp in two axes, wrap in the third,
              linear, no mip filtering                 # the material lookup volume
smp_smap      clamp,  linear, *comparison* sampling with
              a less-or-equal test                     # shadow map depth comparison
smp_jitter    wrap,   nearest, no mip filtering        # a noise texture read unfiltered
```

**Invariants** — These names appear verbatim in the shipped shader sources, so the set and its meanings are part of the frozen data contract. Two of them carry real technique requirements: the shadow sampler performs the depth comparison in the sampler rather than in the shader, and the material sampler wraps in its third axis because the material lookup table is a volume indexed by two clamped surface quantities and one cyclic one.

## `r_dx11Texture`

**Contract** — bind a texture to a named shader resource. Looks the name up in the merged constant table, verifies it really is a texture resource, resolves it to a flat slot and records (slot, texture) on the pass. A name the programs do not declare is silently ignored — a material may mention a texture a particular variant's shader does not read.

**Notes** — In the legacy-dialect compatibility mode the call first tries the *combined* sampler-and-texture path, because in that dialect one name is both. Only if that finds nothing does it fall through to the separate-texture path. That two-step is what lets one material file serve both shader dialects.

Texture names are stripped of any image extension before use, for the same reason as in the loader.

## `r_Stencil` / `r_StencilRef` / `r_CullMode`

**Contract** — record the pass's stencil configuration, reference value and cull mode into the recorded state block.

**Invariants** — The stencil configuration is written to **both winding directions identically**. The engine has no two-sided stencil technique; writing both faces is how the single-sided behaviour of the older generation is reproduced on a device that always has two.

A disabled stencil records only the disable and stops, which is what lets the state cache collapse every stencil-off pass onto one object.

## `i_dx11FilterAnizo`

**Contract** — set or clear the anisotropic-filtering flag for one sampler slot. Separate from the general filter setter because anisotropy is a mode rather than a filter choice, and because its degree is a global setting the sampler cache supplies (see [`StateManager/dx11SamplerStateCache.cpp`](StateManager/dx11SamplerStateCache.cpp.md)).
