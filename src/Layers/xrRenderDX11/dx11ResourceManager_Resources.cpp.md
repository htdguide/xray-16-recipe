# src/Layers/xrRenderDX11/dx11ResourceManager_Resources.cpp

> The backend's deduplicating factory: every pass, program, vertex declaration, constant block and input signature exists exactly once, found by content rather than by name.

**Needs** — [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [`xrRender/ShaderResourceTraits.h`](../xrRender/ShaderResourceTraits.h.md) · [`xrRender/blender_recorder.h`](../xrRender/blender_recorder.h.md) · [`dx11ConstantBuffer.h`](dx11ConstantBuffer.h.md) · [`xrRender/BufferUtils.h`](../xrRender/BufferUtils.h.md) · [`dx11BufferUtils.cpp`](dx11BufferUtils.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Blender_Recorder_R3.cpp`](Blender_Recorder_R3.cpp.md) · [`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) · [`dx11r_constants.cpp`](dx11r_constants.cpp.md) · [`r4_shaders.cpp`](../xrRenderPC_R4/r4_shaders.cpp.md)
**Tier floor** — T1: it compares compiled binary blobs byte for byte and owns device objects whose release order matters.

## Purpose

A level's materials name the same programs, the same vertex layouts and the same state combinations thousands of times over. This file is where that redundancy is collapsed. Every factory function here follows one shape — *search the registry for an equal object, return it if found, otherwise build, register and return* — and the interesting content is what "equal" means for each kind, because that choice decides how much sharing actually happens.

The file also holds the **program name mangling**, which is how a single named program in a material becomes one of several compiled variants.

## `_CreatePass`

**Contract** — interns a render pass: the tuple of (resolved state block, the six stage programs, the constant table, the texture list, the sampler list). Equal passes share one object. Passes are what the draw stream is sorted by, so fewer distinct passes directly means fewer state changes per frame.

## `_CreateVS` — and the variant mangling

**Contract** — creates or finds a vertex program. **The requested name is first mangled with the current skinning mode**: the engine supports several skeletal skinning formulations (by bone count and by weight layout) and each needs a differently compiled program, so the mode index is appended to the name before the lookup.

```text
FUNCTION create_vertex_program(name) -> program
  variant_name = name + "_" + current_skinning_mode     # 0..4
  RETURN create_or_find_shader(variant_name, source_name = name)
```

**Invariants** — The *variant* name is the cache key; the *original* name is what gets compiled, with the variant expressed as a macro. That separation is the whole mechanism: one source file, many compiled objects, distinguished by the macro set. See [`../xrRenderPC_R4/r4_shaders.cpp`](../xrRenderPC_R4/r4_shaders.cpp.md) for the macro set itself.

## `_CreatePS`

**Contract** — same, mangled with the current multisample sample count (zero through seven) instead: a pixel program compiled for per-sample execution differs from one compiled for per-pixel.

## `_CreateGS`, `_CreateHS`, `_CreateDS`, `_CreateCS`

**Contract** — geometry, hull, domain and compute programs, with no mangling — no variant axis applies to them.

## `_DeleteVS`

**Contract** — releases a vertex program and, when that was the last reference, **walks every vertex declaration and destroys the input layout that declaration had memoized for this program's signature**. This is the other half of the layout memoization in [`dx11R_Backend_Runtime.h`](dx11R_Backend_Runtime.h.md): layouts are created lazily per (declaration, signature) pair and would otherwise leak for the life of the process as materials come and go between levels.

## `_CreateDecl`

**Contract** — interns a vertex declaration, comparing by the declaration's element list. On creation it stores both forms: the engine's frozen element list (the one models ship with) and the device's translated form, produced once here. It also owns the per-signature layout map that the draw path fills in.

**Notes** — The previous API generation created a device object from a declaration at this point; here the declaration is only a description, and the device object cannot exist until a program is known. That is the single biggest structural difference between the two generations for a rebuilder, and it is why the memoization exists at all.

## `_CreateConstantBuffer` / `_DeleteConstantBuffer`

**Contract** — interns a constant block, *per submission context*. The equality test is the block's own similarity test — name, layout fingerprint and member names — so two programs declaring the same block share one. Deletion unregisters, reporting loudly if the block was not found, which would mean a double release.

**Notes** — The candidate block must be constructed before it can be compared, so a duplicate costs one construction and one destruction, including a device buffer created and immediately released. That is accepted because it happens only at load time.

## `_CreateInputSignature` / `_DeleteInputSignature`

**Contract** — interns a vertex program's input signature, comparing the compiled blobs **byte for byte**. Since the signature is what layouts are keyed by, sharing it is what lets two vertex programs with identical input requirements share every input layout.

**Notes** — Byte equality is the right test here and a cheaper one would be wrong: a signature is a compiled artifact, and two signatures that differ anywhere describe potentially different input expectations.
