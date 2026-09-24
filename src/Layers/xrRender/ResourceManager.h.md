# src/Layers/xrRender/ResourceManager.h

> The renderer's single resource registry: every texture, material, pass, state block, program and animator in the process is interned here by name or by value, so that identical things are one thing.

**Needs** — [`Shader.h`](Shader.h.md) · [`SH_Texture.h`](SH_Texture.h.md) · [`SH_Matrix.h`](SH_Matrix.h.md) · [`SH_Constant.h`](SH_Constant.h.md) · [`SH_RT.h`](SH_RT.h.md) · [`SH_Atomic.h`](SH_Atomic.h.md) · [`Blender.h`](Blender.h.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`ShaderResourceTraits.h`](ShaderResourceTraits.h.md) · [`tss_def.h`](tss_def.h.md) · [`TextureDescrManager.h`](TextureDescrManager.h.md) · [`xrScriptEngine/script_engine.hpp`](../../xrScriptEngine/script_engine.hpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`Blender.cpp`](Blender.cpp.md) · [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) · [`Blender_Recorder_R2.cpp`](Blender_Recorder_R2.cpp.md) · [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md) · [`ColorMapManager.cpp`](ColorMapManager.cpp.md) · [`D3DXRenderBase.cpp`](D3DXRenderBase.cpp.md) · [`D3DXRenderBase.h`](D3DXRenderBase.h.md) · [`FBasicVisual.cpp`](FBasicVisual.cpp.md) · [`FSkinned.cpp`](FSkinned.cpp.md) · [`R_Backend_DBG.cpp`](R_Backend_DBG.cpp.md) · [`R_DStreams.cpp`](R_DStreams.cpp.md) · [`ResourceManager.cpp`](ResourceManager.cpp.md) · [`ResourceManager_Loader.cpp`](ResourceManager_Loader.cpp.md) · [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md) · _and 29 more_
**Tier floor** — T1: it is the owner of every device-allocated object in the renderer, and the order in which those objects are released and rebuilt across a device reset is part of its contract.

## Purpose

This header is the exception to the recipe's usual rule that a `.cpp` carries the substance: there is no `ResourceManager.cpp` that owns this type. The implementation is spread over five files that were split by *topic*, not by layering, and two more that live in the graphics backends. So the **shared state and its invariants live here**, and each implementation file's page describes only its own algorithms.

Where the pieces are:

| File | What it decides |
|---|---|
| [`ResourceManager.cpp`](ResourceManager.cpp.md) | Material compilation: name → blender → pass list. Name-list parsing, element/shader interning, memory reporting. |
| [`ResourceManager_Loader.cpp`](ResourceManager_Loader.cpp.md) | The shipped material library file format, and device create/destroy. |
| [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md) | The device-lost path: what is released, what is rebuilt, and in what order. |
| [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) | The interning primitives — one create/delete pair per resource kind. |
| [`ResourceManager_Scripting.cpp`](ResourceManager_Scripting.cpp.md) | The scripted material definitions the game data ships, and the declarative surface they are written against. |
| [`ShaderResourceTraits.h`](ShaderResourceTraits.h.md) | The generic machinery that compiles, caches and creates one *program* per pipeline stage. |
| Backend layers (chapters 20, 21) | The per-API bodies of the program creators — the same names, filled twice. |

The split is historical and partly arbitrary; a rebuild is free to merge the first five. What is *not* arbitrary is the boundary to the last two: program creation is the only part of this type whose body differs per graphics API, and it is declared here and defined there deliberately, so that the rest of the renderer never learns which backend is running.

## State

One instance exists per process, reachable through the global environment struct (chapter 5).

```text
RECORD ResourceManager

  # --- interned by name: exactly one object per distinct name ---
  blenders        : map<text, Blender>        # material templates, from the shipped library
  textures        : map<text, Texture>
  matrices        : map<text, MatrixAnimator>     # animated texture transforms
  constants       : map<text, ConstantAnimator>   # animated scalar/colour constants (oldest generation only)
  render_targets  : map<text, RenderTarget>
  programs_vertex, programs_pixel, programs_geometry,
  programs_hull, programs_domain, programs_compute : map<text, Program>
  programs_linked : map<text, LinkedProgramPipeline>   # only where the API links stages into one object
  texture_details : map<text, (detail_texture_name, constant_setup)>

  # --- interned by value: a new one is created only if no equal one exists ---
  states          : list<StateBlock>          # rasterizer/blend/depth/sampler, as one unit
  declarations    : list<VertexDeclaration>
  geometries      : list<Geometry>            # (declaration, vertex buffer, index buffer, stride)
  constant_tables : list<ConstantTable>       # name -> register binding, per program
  texture_lists   : list<TextureList>         # (stage, texture) pairs, sorted by stage
  matrix_lists    : list<MatrixList>
  constant_lists  : list<ConstantList>
  passes          : list<Pass>
  elements        : list<ShaderElement>       # an ordered pass list plus its sort flags
  shaders         : list<Shader>              # six elements: one per render purpose

  # --- misc ---
  texture_descriptions : TextureDescriptions  # detail-texture and bump pairings, from shipped data
  constant_setups      : list<(name, setup_callback)>  # engine-side suppliers of named constants
  necessary_textures   : list<Texture>        # pinned across level transitions
  deferred_load        : bool                 # textures are created but not uploaded until asked
  shader_fallback_allowed : bool              # a missing material is survivable, not fatal
  script_vm            : ScriptVM             # its own interpreter, not the game's
  shaders_lock, script_vm_lock : locks
```

**Invariants**

- **Name interning is by pointer identity after the fact.** The map's key is the resource's *own* name string, handed over when the resource is created; deleting a resource looks it up by its own name. The consequence a rebuild must honour: a resource may not be renamed after registration, because the map's key would no longer be the name the delete path searches for.
- **Every registered resource carries a "registered" flag,** and `delete` returns immediately when it is absent. The flag distinguishes objects the registry owns from stack-built prototypes that were never installed — the value-interning creators are always handed a prototype by value, and that prototype must be destroyed without disturbing the registry. This is the single mechanism that makes "create by value, intern by equality" safe.
- **Reference counting is the only lifetime rule.** The registry holds *no* reference of its own: an entry drops out when its last external reference goes. The exceptions are the animator maps, which are created with one reference the registry keeps — because they are loaded eagerly from the material library and must survive until teardown, whether or not a material uses them.
- **Deleting a resource the registry does not know about is an error that is logged, not raised.** A renderer that lost track of a resource is already wrong; the shipped engine reports and continues, because a crash at teardown loses the log.
- **Empty means none.** A constant table with no entries, a matrix list whose every slot is empty, a constant list likewise, an element with no passes — all intern to *nothing* rather than to an empty object. Downstream code tests the absence, so an empty object and a missing object must not both exist.
- **Texture lists are sorted by stage before comparison.** Two materials binding the same textures to the same stages in a different authoring order are the same list. Without the sort the interning would miss, and the pass count would inflate.
- The texture name `null` and the animator names `$null` (case-insensitively) resolve to nothing, not to a resource called "null". These are the frozen spellings the shipped material data uses for an unfilled slot.
- The script interpreter here is **separate from the game's**. It shares the language seam but not the state: material scripts run at load time in their own virtual machine, with a lock around it because materials are compiled from several threads.

## Exported units

**Material creation** — the surface the rest of the renderer uses.

- `Create` — the main entry: a material name plus comma-separated texture, constant and matrix name lists, yielding a shader. Two overloads: by name, and against a caller-supplied blender.
- `Delete` — drop a shader from the registry.
- `_cpp_Create` · `_lua_Create` · `_lua_HasShader` — the two compilation paths and the test that chooses between them. See [`ResourceManager.cpp`](ResourceManager.cpp.md) and [`ResourceManager_Scripting.cpp`](ResourceManager_Scripting.cpp.md).
- `_ParseList` — split a comma-separated name list into normalized names.
- `_GetBlender` · `_FindBlender` · `_GetBlenders` — look up a material template; the first reports a miss, the second does not.
- `RegisterConstantSetup` — register an engine-side supplier for a named shader constant.

**Lifecycle**

- `OnDeviceCreate` — load the material library, from a path or an open reader.
- `OnDeviceDestroy` — release the library, animators and blenders.
- `reset_begin` · `reset_end` — the device-lost bracket. See [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md).
- `CompatibilityCheck` — sniff the shipped shader sources for a known variant.
- `DeferredLoad` · `DeferredUpload` · `DeferredUnload` — batch texture upload, so a level's textures are decoded together rather than one per material.
- `StoreNecessaryTextures` · `DestroyNecessaryTextures` — pin the non-level textures across a level transition.
- `Evict` — reclaim device memory under pressure; a no-op in the shipped build.

**Interning primitives** — a create/delete pair per kind, all described in [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md).

- `_CreateTexture` · `_DeleteTexture`
- `_CreateMatrix` · `_DeleteMatrix` — animated texture transform
- `_CreateConstant` · `_DeleteConstant` — animated constant, oldest generation only
- `_CreateRT` · `_DeleteRT` — render target, by name plus dimensions, format, sample count and slice count
- `_CreateState` · `_DeleteState` — a recorded block of rasterizer/blend/depth/sampler settings
- `_CreateDecl` · `_DeleteDecl` — vertex declaration
- `_CreateConstantTable` · `_DeleteConstantTable`
- `_CreateTextureList` · `_CreateMatrixList` · `_CreateConstantList` and their deletes
- `_CreateElement` · `_DeleteElement` — an ordered pass list with its sort flags
- `_CreatePass` (backend) · `_DeletePass`
- `CreateGeom` · `DeleteGeom` — bind a declaration to a vertex and index buffer; a second overload takes the legacy packed vertex-format word instead of a declaration
- `_CreateConstantBuffer` · `_CreateInputSignature` and their deletes — only on the backend that has those concepts

**Program creation** — declared here, bodies in the backends.

- `_CreateVS` · `_CreatePS` · `_CreateGS` · `_CreateHS` · `_CreateDS` · `_CreateCS` and their deletes — one per pipeline stage.
- `_CreatePP` · `_LinkPP` · `_DeletePP` — where the API requires the stages be linked into one pipeline object before use.
- `GetShaderMap` · `CreateShader` · `DestroyShader` — the generic bodies that all of the above delegate to, parameterized by stage. See [`ShaderResourceTraits.h`](ShaderResourceTraits.h.md).

**Authoring-tool cooperation**

- `ED_UpdateBlender` · `ED_UpdateMatrix` · `ED_UpdateConstant` · `ED_UpdateTextures` — replace a registered definition in place so an edit is seen without a restart. Replacing a blender asserts the class identifier is unchanged: the parameter block's layout belongs to the class, so swapping classes under a live material would reinterpret bytes.

**Diagnostics**

- `Dump` — counts and per-resource reference counts for every map and list.
- `_GetMemoryUsage` · `_DumpMemoryUsage` — texture bytes, split into lightmaps and everything else.
- `DBG_VerifyGeoms` · `DBG_VerifyTextures` — assert the maps' keys still agree with their values' names.

**Notes**

- The `reclaim` helper — a linear scan that removes a pointer from a list — is the delete half of every value-interned kind. It is linear over a list of a few thousand at teardown and at material-rebuild time only, never per frame. That it is linear is an accepted cost, not an oversight; a rebuild with a better container loses nothing.
- Several kinds are declared per-context (constant buffers) on one backend and not at all on another. The *kind* is what survives a rebuild; the per-context multiplicity is a fact about that API's threading model and belongs to the backend chapter.
