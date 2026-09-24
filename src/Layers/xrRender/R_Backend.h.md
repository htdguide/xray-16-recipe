# src/Layers/xrRender/R_Backend.h

> The render command list: a redundancy-filtering mirror of the whole graphics device state, through which every draw in the engine passes.

**Needs** — [`R_DStreams.h`](R_DStreams.h.md) · [`R_Backend_xform.h`](R_Backend_xform.h.md) · [`R_Backend_hemi.h`](R_Backend_hemi.h.md) · [`R_Backend_tree.h`](R_Backend_tree.h.md) · [`Shader.h`](Shader.h.md) · [`SH_Texture.h`](SH_Texture.h.md) · [`SH_RT.h`](SH_RT.h.md) · [`r_constants_cache.h`](r_constants_cache.h.md) · [`BufferUtils.h`](BufferUtils.h.md) · [`FVF.h`](FVF.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md) · [`BufferUtils.h`](BufferUtils.h.md) · [`dxPixEventWrapper.h`](Debug/dxPixEventWrapper.h.md) · [`DetailManager.h`](DetailManager.h.md) · [`DetailManager_soft.cpp`](DetailManager_soft.cpp.md) · [`FProgressive.cpp`](FProgressive.cpp.md) · [`FVisual.cpp`](FVisual.cpp.md) · [`ParticleEffect.cpp`](ParticleEffect.cpp.md) · [`R_Backend.cpp`](R_Backend.cpp.md) · [`R_Backend_DBG.cpp`](R_Backend_DBG.cpp.md) · [`R_Backend_Runtime.cpp`](R_Backend_Runtime.cpp.md) · [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md) · [`R_Backend_hemi.cpp`](R_Backend_hemi.cpp.md) · [`R_Backend_hemi.h`](R_Backend_hemi.h.md) · _and 33 more_
**Tier floor** — T1: it caches raw device handles and byte strides, and its whole reason to exist is that a redundant state change costs more than the comparison that avoids it, so every setter must be inlined into the caller's hot loop.

## Purpose

Nothing in the engine talks to the graphics device directly. Everything talks to a **command list** — one of these — and the command list decides whether a device call actually happens. It owns a shadow copy of every piece of device state the engine ever sets: bound render targets, vertex and index buffers, the vertex layout, each shader stage's program, the texture bound at each stage, the depth/stencil/blend/raster fields, and the named-constant cache. A setter compares against the shadow and returns without touching the device when nothing changed.

That is the entire design. The renderer's draw stream (chapter 18's sorted draw items) is built so that consecutive items share as much state as possible; this class is what converts that sorting effort into saved work. The per-class counters it keeps are the feedback loop: they say how many state changes the sort *failed* to avoid.

A second, less obvious job: the command list is also the **recording target**. More than one can exist — one bound to the device's immediate submission path and others recording into deferred buffers — so the visible-geometry walk can be split across worker threads and each thread's recorded commands replayed in order on the immediate one. `context_id` names which of those a given command list is.

## State

```text
RECORD CommandList
  # --- named-constant sub-caches (see their own files) ---
  xforms        : TransformCache   # world/view/projection and their products
  hemi          : HemiCache        # per-object ambient-cube and material constants
  tree          : TreeCache        # wind/wave constants for animated vegetation
  lod           : LodCache         # level-of-detail blend constants (D3D11 path only)

  # --- bound targets ---
  render_targets : list<TargetHandle>   # exactly 4 slots
  depth_target   : TargetHandle
  rt_width       : int                  # dimensions of the currently bound target set,
  rt_height      : int                  # published so passes can size their viewports

  # --- geometry ---
  layout        : VertexLayout          # the declaration, not the buffer
  vertex_buffer : BufferHandle
  index_buffer  : BufferHandle
  vertex_stride : int (bytes)           # invariant: matches `layout`; the device is told
                                        # the stride separately from the layout

  # --- programs ---
  program_per_stage : map<Stage, ProgramHandle>   # vertex, pixel, geometry,
                                                  # hull, domain, compute
  pipeline      : ProgramHandle         # the linked set, where the device has that concept
  constants     : ConstantCache         # the value side; see r_constants_cache
  constant_table: ConstantTable         # the name→location side of the *current pass*

  # --- fixed state, each an integer mirroring one device field ---
  state_block   : StateHandle           # the compiled blend/depth/raster/sampler unit
  stencil_enable, stencil_func, stencil_ref, stencil_mask,
  stencil_writemask, stencil_fail, stencil_pass, stencil_zfail : int
  colorwrite_mask, fill_mode, cull_mode, z_enable, z_func, alpha_ref : int

  # --- resource lists of the current pass ---
  textures      : TextureList           # the pass's (stage, texture) list
  matrices      : MatrixList            # the pass's animated texture transforms
  constant_list : ConstantList          # the pass's animated colour constants

  # --- expanded binding, one entry per stage slot ---
  bound_texture : map<Stage, list<Texture>>
  bound_matrix  : list<Matrix>          # 8 slots, fixed-function texture transforms only

  # --- per-object lighting latched before a draw ---
  ambient        : real                 # hemispherical term for this object
  ambient_cube   : list<real>           # 6 faces, order +X +Y +Z -X -Y -Z
  sun            : real                 # sun visibility term for this object

  stat          : Statistics
  context_id    : int                   # which submission context this list records into
```

**Invariants**

- Every cached field holds either a value the device currently has, or the **impossible sentinel** (all bits set). Invalidation writes the sentinel everywhere rather than a plausible default, because a plausible default would make the *next* set look redundant and silently skip a real device call. This is the single most important rule in the file: a rebuild that initialises these to zero will drop the first draw's state.
- `rt_width`/`rt_height` describe the currently bound target set and are read by passes rather than recomputed; they must be updated in the same operation that binds targets.
- `ambient`, `ambient_cube` and `sun` are written by whoever is about to draw an object and consumed by `apply_lmaterial`; they are a parameter-passing channel, not persistent state.

```text
RECORD Statistics
  render  : { calls, verts, polys }
  compute : { calls, groups_x, groups_y, groups_z }
  changes : map<Kind, int>    # one counter per state kind: programs (per stage),
                              # layout, vertex buffer, index buffer, state blocks,
                              # textures, matrices, constants, transforms, targets
  by_class : map<GeometryClass, { verts, draw_calls }>
```

`GeometryClass` is a closed set and is worth naming exactly, because it is the engine's own taxonomy of the draw stream and the performance overlay reports it verbatim:

```text
ENUM GeometryClass
  static          # level geometry
  flora           # wind-animated vegetation at full detail
  flora_lods      # the impostor form of the same
  details         # the grass/debris layer
  ui
  dynamic         # skinned models, generic
  dynamic_sw      # skinned on the processor rather than in the vertex program
  dynamic_inst    # instanced dynamic geometry
  dynamic_1B      # skinned with 1 bone influence per vertex
  dynamic_2B      # ... 2 influences
  dynamic_3B      # ... 3
  dynamic_4B      # ... 4
```

## Texture stage numbering

**Contract** — Textures are addressed by a single flat integer across *all* shader stages, not by (stage, slot) pairs. The number space is partitioned into per-stage blocks and the command list decodes an incoming number back into (stage, slot) when it binds. The partition is a frozen convention, because pass descriptions loaded from the shipped material files name stages by these numbers.

```text
ENUM StageBase                # base index of each stage's block
  pixel    = 0
  vertex   = <first vertex block>
  geometry = vertex   + 256
  hull     = geometry + 256
  domain   = hull     + 256
  compute  = domain   + 256
  invalid  = compute  + 256
```

**Notes** — The 256 spacing is not the slot count; the per-stage slot counts are 16 (pixel, geometry, hull, domain, compute) and 4 (vertex). 256 is chosen because the device generation in use allows up to 128 distinct bindings per stage and the gap must exceed that, so a decoder can recover the stage by integer division without a table. On the device where all stages share one texture namespace, the blocks are instead packed tight (each stage's base is the previous base plus its slot count), and the same flat numbers still decode — which is why the decode must be a function and not arithmetic hard-coded at call sites.

## `set_Shader` / `set_Element` / `set_Pass`

**Contract** — The three form a funnel from the material system into the device. A *shader* (in this project's sense: a compiled material, not a device program) holds up to six *elements*, one per rendering situation; an element holds up to two *passes*; a pass is the smallest unit that can be applied. `set_Shader(S, p)` applies element 0's pass `p`; `set_Element(E, p)` applies `E`'s pass `p`; `set_Pass` does the work.

```text
FUNCTION set_Pass(pass)
  set_state_block(pass.state)      # blend, depth, raster, samplers as one unit
  IF device has a linked pipeline object AND pass.pipeline is present
    bind pipeline                  # one call replaces the per-stage binds
  ELSE
    FOR EACH stage IN present stages OF pass
      bind pass.program[stage]
  set_constant_table(pass.constants)   # name→location table for the *whole pass*
  set_textures(pass.textures)
  set_matrices(pass.matrices)
```

**Notes** — The order matters in one place only: the constant table must be installed before any code looks a constant up by name, because name lookups go through the *currently installed* table and silently do nothing when there is none. Everything else in the pass is order-independent.

## `set_Textures`

**Contract** — Installs a whole pass's texture list at once. For each (stage-number, texture) entry it decodes the stage, compares against the shadow, and binds through the texture's own bind hook when it differs. It then **clears every slot above the highest one this list touched**, per stage.

**Invariants** — After the call, no slot holds a texture that is not in the list. That trailing clear is load-bearing: leaving a stale texture bound in a slot the current program does not read is harmless on some devices and a validation error or a hazard (the same resource bound for reading and writing) on others.

A texture also re-binds when its *slice selector* changed even though the texture object did not, which is how array textures address a different layer without a new object.

**Notes** — Whether a texture list is identical to the one already installed is deliberately *not* used as an early-out. Two passes can share a texture list object yet need different views of it, so the comparison is done per slot.

## `Invalidate`

**Contract** — Drops every cached assumption: clears bound targets, buffers, programs, lists and expanded slots; sets every fixed-state field to the impossible sentinel; unmaps all named-constant bindings in the sub-caches; and returns the command list to the immediate context. Costs nothing but the next frame's first state sets.

**Notes** — Called on device creation, at both ends of a frame, and after anything that could have changed device state behind the engine's back. Unmapping the constant sub-caches is not optional housekeeping: those caches hold *locations* resolved against the previously installed pass, and a location is meaningless once the pass changes.

## `OnFrameBegin` / `OnFrameEnd`

**Contract** — `OnFrameBegin` invalidates, binds the base colour and depth targets, zeroes the statistics and disables stencil. `OnFrameEnd` invalidates again, after asking the device to forget its own state where the device offers that. Both are no-ops when the process runs as a dedicated server with no graphics device.

## `apply_lmaterial`

**Contract** — Publishes the per-object lighting values latched in `ambient`, `ambient_cube` and `sun` into the current pass's constants, together with the *material index* of the texture bound at the pass's base sampler. Does nothing if the current pass declares no base sampler.

```text
FUNCTION apply_lmaterial()
  c = lookup_constant("s_base")        # the base sampler, by frozen name
  IF c is none THEN RETURN
  texture = bound_texture_at(c.sampler_index)
  material = texture.material          # a small integer from the texture description data
  hemi.set_material(ambient, sun, 0, (material + 0.5) / 4)
  hemi.set_pos_faces(ambient_cube[+X], ambient_cube[+Y], ambient_cube[+Z])
  hemi.set_neg_faces(ambient_cube[-X], ambient_cube[-Y], ambient_cube[-Z])
```

**Notes** — `(material + 0.5) / 4` is a texture-coordinate computation, not a scale: the material index selects one of four rows in a lookup texture that holds the shipped lighting response curves, and the half-step lands the sample in the middle of its row rather than on the boundary between two. The divisor is therefore tied to "four material rows" and must move with it.

## `Render`

**Contract** — Two forms: indexed (base vertex, first vertex, vertex count, first index, primitive count) and non-indexed (first vertex, primitive count). Both apply any deferred binding the device needs (the vertex layout, which on some devices must be matched against the current vertex program's signature; the primitive topology; a pending target change), issue the draw, and add to the statistics.

**Notes** — Binding the layout *at draw time* rather than when the buffer is set is deliberate on devices where a layout object is valid only for a particular vertex-program signature: the layout cannot be resolved until both the buffer and the program are known, and draw time is the first moment both are certain.

## `submit`

**Contract** — For a command list recording into a deferred context: closes the recording and replays it on the immediate context. Fails a check if called on the immediate list. A no-op where the device has no deferred recording.

## `set_Stencil` / `set_Z` / `set_ZFunc` / `set_AlphaRef` / `set_ColorWriteEnable` / `set_CullMode` / `set_FillMode` / `set_Scissor` / `SetViewport`

**Contract** — One shadowed field each (stencil sets eight at once); each writes to the device only on change. `set_Stencil` with stencil disabled ignores the remaining parameters.

## `set_Vertices` / `set_Indices` / `set_Geometry` / `set_Format`

**Contract** — `set_Geometry` is the normal entry point: it takes a geometry record (layout + vertex buffer + index buffer + stride) and applies all four. The individual setters exist for code that builds geometry on the fly into the dynamic streams.

## `set_Constants` / `get_c` / `set_c` / `set_ca`

**Contract** — `set_Constants` installs the pass's name→location table. `get_c(name)` resolves a name against the installed table and yields a location handle, or nothing when no table is installed or the name is absent — an absent name is *not* an error, because one pass list is shared across material variants that use different subsets. `set_c` writes a value at a location; `set_ca` writes an array element. Both accept either a resolved location (fast, and what per-frame code should hold) or a name (resolved on each call). Writing through a null location is a silent no-op, which is what makes "set everything the engine knows about and let the pass ignore what it does not want" a legitimate pattern throughout the renderer.

## `set_pass_targets` / `set_RT` / `set_ZB` / `get_RT` / `get_ZB` / `ClearRT` / `ClearZB` / `ClearRTRect` / `ClearZBRect`

**Contract** — Target binding is shadowed like everything else; up to four colour targets plus depth. `set_pass_targets` binds three colour targets and a depth target as one unit, which is the shape the deferred-shading path needs. Clears take either a target handle or a target resource, optionally restricted to a list of rectangles; the rectangle forms report whether the device could honour them.

**Notes** — On a device where the depth target is per-context (because several command lists record concurrently), the depth resource carries one target view per context and the clear picks by `context_id`. That is the only place the context identity leaks into resource selection.

## `get_ActiveTexture`

**Contract** — Given a flat stage number, yields the texture currently bound there. Fails a check on a number outside the partition.

## `gpu_mark_begin` / `gpu_mark_end`

**Contract** — Push and pop a named region in the device's own capture timeline. Debug aid; may be empty.

## `Compute`

**Contract** — Dispatches a compute program over a three-dimensional group count, applying any pending binding first, and adds to the compute statistics. Present only where the device has compute.

## `OnDeviceCreate` / `OnDeviceDestroy` / `SetupStates`

**Contract** — `OnDeviceCreate` acquires the debug-annotation channel, builds the two debug-draw geometries, and invalidates. `OnDeviceDestroy` releases them. `SetupStates` applies the settings that are global rather than per-pass: the default winding, the anisotropy limit and the mip bias, all read from console variables.

## `dbg_*`

**Contract** — The debug-draw surface: immediate line, triangle, box, oriented box and ellipse drawing through the dynamic streams, plus the overdraw visualisation bracket. See [`R_Backend_DBG.cpp`](R_Backend_DBG.cpp.md).

**Notes** — This family is compiled away in a shipping build except for the two geometry-submitting helpers and the overdraw bracket. A rebuild may drop it wholesale; nothing in the rendered image depends on it.
