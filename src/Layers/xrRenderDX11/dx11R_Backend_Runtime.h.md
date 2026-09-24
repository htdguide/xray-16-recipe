# src/Layers/xrRenderDX11/dx11R_Backend_Runtime.h

> The backend half of the render command list: how a state change, a constant-table swap and a draw actually reach the device on this filling of the graphics-device seam.

**Needs** — [`xrRender/R_Backend.h`](../xrRender/R_Backend.h.md) · [`StateManager/dx11ShaderResourceStateCache.h`](StateManager/dx11ShaderResourceStateCache.h.md) · [`StateManager/dx11StateManager.h`](StateManager/dx11StateManager.h.md) · [`dx11HW.h`](dx11HW.h.md) · [`dx11r_constants_cache.h`](dx11r_constants_cache.h.md) · [`CommonTypes.h`](CommonTypes.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx113DFluidEmitters.cpp`](3DFluid/dx113DFluidEmitters.cpp.md) · [`dx113DFluidGrid.cpp`](3DFluid/dx113DFluidGrid.cpp.md) · [`dx113DFluidManager.cpp`](3DFluid/dx113DFluidManager.cpp.md) · [`dx113DFluidObstacles.cpp`](3DFluid/dx113DFluidObstacles.cpp.md) · [`dx113DFluidRenderer.cpp`](3DFluid/dx113DFluidRenderer.cpp.md) · [`dx11DetailManager_VS.cpp`](dx11DetailManager_VS.cpp.md) · [`r4_rendertarget_accum_direct.cpp`](../xrRenderPC_R4/r4_rendertarget_accum_direct.cpp.md) · [`r4_rendertarget_phase_hdao.cpp`](../xrRenderPC_R4/r4_rendertarget_phase_hdao.cpp.md) · [`r4_rendertarget_u_set_rt.cpp`](../xrRenderPC_R4/r4_rendertarget_u_set_rt.cpp.md)
**Tier floor** — T1: every operation here is on the hot path of every draw in the engine, and its entire value is that a comparison is cheaper than a device call, so each one must inline into its caller.

## Purpose

Chapter 18 declares a *command list* — a shadow copy of all device state with a redundancy filter in front of it. This file supplies that declaration's bodies for this backend. It is a header with no implementation file because every function in it must be inlined: these are the innermost calls of the frame.

Three ideas here are the ones a rebuild must reproduce, and none of them is about any particular graphics API:

1. **Setters record, draws bind.** A setter updates the shadow and, for most state, defers the actual device work. Nothing is bound until a draw or a dispatch demands it, and then everything is bound in one fixed order.
2. **The vertex layout is a function of two things, not one.** A vertex declaration alone cannot produce a layout object; the device validates the layout against the *input signature of the bound vertex program*. So the layout is memoized per (declaration, signature) pair.
3. **Constants are bound as whole tables, per stage, and the swap is diffed.** A material pass hands over one constant table; the backend distributes its buffers into per-stage slot arrays, rebinds only the contiguous range that changed, and then runs the *by-name* constant handlers.

## State

The record lives in chapter 18's declaration; the fields this file reads and writes are:

```text
RECORD CommandListDeviceState        # the backend-owned part
  context_id        : int            # which submission context this list records into
  render_targets    : list<handle>   # 4 colour slots (only 3 are ever set by a pass)
  depth_target      : handle
  targets_dirty     : bool           # set by any target change, cleared at the next draw
  curr_rt_width     : int            # dimensions of the current target set; the viewport
  curr_rt_height    : int            # and several shader constants are derived from these
  decl              : VertexDeclaration
  input_signature   : handle         # input signature of the currently bound vertex program
  input_layout      : handle         # the memoized (decl, signature) product currently bound
  vertex_buffer     : handle
  vertex_stride     : int            # part of the comparison key: same buffer, new stride, rebind
  index_buffer      : handle
  topology          : enum
  programs          : per-stage handle  # vertex, pixel, geometry, hull, domain, compute
  constants_by_stage: per-stage list<ConstantBuffer>   # fixed number of slots per stage
  ctable            : ConstantTable  # the table currently bound; identity is the cache key
```

Invariant: `targets_dirty` implies the device currently has **no** targets bound (see `set_RT`). Invariant: a draw requires a bound vertex program, a declaration and a signature — all three, or the layout cannot be produced.

## `Render` — the draw path

**Contract** — issues one indexed or non-indexed draw of a primitive run. Accumulates per-frame counters (calls, vertices, polygons) that the renderer's statistics display reads. Everything deferred by the setters is resolved here.

```text
FUNCTION render(primitive_kind, base_vertex, start_vertex, vertex_count, start_index, primitive_count)
  topology = translate(primitive_kind)
  index_count = indices_for(primitive_kind, primitive_count)
    # point: n ; line list: 2n ; line strip: n+1 ; triangle list: 3n ; triangle strip: n+2
    # triangle fan has no translation: the engine is expected to stop emitting it

  IF a tessellation stage is bound THEN
    REQUIRE topology is triangle list
    topology = three-control-point patch list   # see Notes

  apply_topology_if_changed(topology)
  bind_shader_resources(context_id)   # the texture/buffer view cache flushes its dirty slots
  bind_targets_if_dirty()
  bind_vertex_layout()                # memoized (declaration, signature) product
  apply_state_blocks()                # rasterizer / depth-stencil / blend / samplers
  flush_constants()                   # LAST: applying state may itself write constants
  draw_indexed(index_count, start_index, base_vertex)
```

**Invariants** — The order of the last five steps is the contract, and only the final one is subtle: the state layer emulates some fixed-function behaviour with shader constants, so a state application can dirty a constant. Flushing constants before applying state would publish stale values for exactly one draw.

The non-indexed variant is the same sequence without the index buffer, and it *silently drops* triangle fans rather than failing — some legacy debug geometry still asks for them.

## `Compute`

**Contract** — dispatches a compute grid of the given thread-group counts. Same preamble as a draw minus everything geometric: shader resources, then state, then constants, then dispatch. Records the group counts into the statistics record. Only available when the device reported compute support; callers check first.

## `ApplyVertexLayout`

**Contract** — ensures the device has an input layout matching the current declaration *and* the current vertex program's input signature. Creates and memoizes one per pair, inside the declaration. Asserts all three inputs are present.

```text
FUNCTION bind_vertex_layout()
  layout = decl.layout_for(input_signature)      # map keyed by signature identity
  IF none THEN
    layout = device.create_input_layout(decl.element_list, input_signature)
    decl.layout_for(input_signature) = layout
  IF layout != input_layout THEN
    input_layout = layout
    device.set_input_layout(layout)
```

**Notes** — This is the single most transferable idea in the file for a rebuilder on a modern API. The declaration is loaded from model data and is API-independent; the signature comes from the compiled program. The product is device-specific and cached on the declaration, so it is created at most once per (mesh format, program) pair over the process's life and is destroyed with the declaration. On an API where a pipeline object subsumes the layout, the same memoization key still applies — the key just names more state.

## `set_Constants`

**Contract** — binds a whole constant table, or unbinds when given nothing. Returns immediately if the same table is already bound: the table's identity *is* the cache key, and material passes are sorted so that consecutive draws share one. On any change it first invalidates every cached mapped pointer (the transform, ambient-cube, vegetation and level-of-detail sub-caches, plus the state layer's), because those pointers address storage inside the previous table's buffers.

```text
FUNCTION set_constants(table)
  IF table == ctable THEN RETURN
  ctable = table
  unmap_all_constant_subcaches()          # their write pointers are now stale
  IF table is none THEN RETURN

  # Distribute this table's buffers into per-stage slot arrays.
  # A buffer's key packs (stage, slot): high bits name the stage, low bits the slot.
  new_slots = all stages, all slots := none
  FOR EACH (key, buffer) IN table.buffers_for(context_id)
    new_slots[stage_of(key)][slot_of(key)] = buffer

  # Rebind only what moved, and only as one contiguous run per stage.
  FOR EACH stage
    IF new_slots[stage] differs from current THEN
      first = lowest differing slot ; last = highest differing slot
      device.bind_constant_buffers(stage, first .. last)
    current[stage] = new_slots[stage]

  # By-name constants that are computed rather than stored (see Notes)
  FOR EACH c IN table.constants
    IF c.handler EXISTS THEN c.handler.setup(this_command_list, c)
```

**Notes** — Two decisions deserve stating plainly.

*Why the contiguous-range diff.* Binding one slot at a time costs a device call per slot per stage per material change; binding all slots every time costs the same. Comparing and binding the single run from the first to the last difference collapses the common case — one buffer changed, or none — into zero or one call.

*The constant handlers are the material system's by-name binding.* Chapter 18's blender recorder turns a material's text description into a constant table whose entries are matched by *name* against the compiled program's reflected constants (see [`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md)). Most entries are plain storage the renderer writes into. Some carry a *handler*: a named quantity the engine computes at bind time — the sun direction, the fog parameters, a time value, a texture's dimensions. Binding a table runs every handler once. The name-to-slot resolution happened once, when the program was compiled and reflected; from then on the binding is by offset. A rebuild must keep exactly this split: names are resolved once at load, and the per-draw path never sees a string.

## `set_RT` / `set_ZB` / `ApplyRTandZB` / `set_pass_targets`

**Contract** — record a colour target in one of four slots, or the depth target, and mark the target set dirty. A change *immediately unbinds all targets on the device* and defers the rebind to the next draw. `set_pass_targets` is the form a render pass actually uses: up to three colour targets plus depth, and it also derives the current target dimensions and sets the viewport to the full target.

**Invariants** — A texture may not be simultaneously a render target and a shader input. Unbinding at record time rather than at draw time is what makes "render to a target, then sample it in the next pass" legal without the caller doing anything: by the time the next pass binds that texture as an input, the device no longer holds it as an output.

**Notes** — The viewport is *always* the whole target; there is no partial-viewport rendering in this engine. Scissoring is used instead for restricted regions, because a scissor does not change the projection.

## `set_Vertices` / `set_Indices` / `set_Geometry` / `set_Format`

**Contract** — bind a vertex buffer with its byte stride, an index buffer, or both plus the declaration in one call from a geometry record. Each is redundancy-filtered. The vertex comparison includes the stride: the same buffer bound with a different stride is a different binding.

**Notes** — **Indices are always 16-bit** on this backend. That is a hard constraint on every geometry producer in the engine: a single draw's vertex range must fit in 65,536 entries, which is why level geometry is split into subsets and why the dynamic geometry streams flush at that bound.

## `set_Scissor` / `SetViewport`

**Contract** — set or clear the scissor rectangle. Clearing sets the rectangle count to zero *and* disables the scissor bit in rasterizer state, which lives in the state layer, so this is one of the few operations that spans both. The viewport is set directly, without redundancy filtering, because it changes only at pass boundaries.

## `set_VS` / `set_PS` / `set_GS` / `set_HS` / `set_DS` / `set_CS`

**Contract** — bind a program to one stage, filtered against the shadow. Binding a vertex program also records its **input signature**, which the layout step needs; that is why the vertex setter has a second form taking the engine's shader record rather than a raw handle. Debug builds also record the program's name for the frame inspector.

`is_TessEnabled` answers whether tessellation is both available (the device granted the capability tier) and currently in use (a hull or domain program is bound).

**Notes** — The tessellation path forces the topology to a three-control-point patch list and requires triangle-list input. The source calls this a hack, and it is: it means tessellation can only ever be applied to indexed triangle lists, and a caller that sets a strip topology while a hull program is bound trips an assertion rather than being converted.

## `set_Stencil`, `set_Z`, `set_ZFunc`, `set_CullMode`, `set_FillMode`, `set_ColorWriteEnable`, `set_AlphaRef`

**Contract** — individual state fields. Each forwards to the state layer, which accumulates them into a description and resolves the description to a cached state object at draw time (see [`StateManager/dx11StateManager.cpp`](StateManager/dx11StateManager.cpp.md)). The cull mode is additionally shadowed here, because the rendering code reads back the current cull mode to decide whether to flip a shadow pass.

**Notes** — `set_AlphaRef` is *not implemented* and asserts if called: this backend has no fixed-function alpha test, and alpha cut-out is expressed in the shader instead. `SetTextureFactor` and `SetAmbient` are likewise inert — both are fixed-function leftovers from the oldest backend that the shared code still calls unconditionally. `set_xform` counts a statistic and does nothing: transforms are constants here, not device state.

## `ClearRT` / `ClearZB` / `ClearRTRect` / `ClearZBRect`

**Contract** — clear a whole target to a colour, or a depth target to a depth (and optionally a stencil) value. The rectangle-limited variants are *optional*: they require a device generation that offers them and return a failure flag when unavailable, so the caller can fall back to a full clear or to drawing a quad. A rebuild must either provide partial clears or keep the caller's fallback.

## `get_ConstantDirect`

**Contract** — look a constant up by name and hand back raw write pointers into the vertex, geometry and pixel storage for it, or nothing for each stage where it is absent. This is the escape hatch used by code that wants to write a large constant block without going through the typed setters — the fluid subsystem and the post-process passes use it. The name lookup is per call, so callers hold the result rather than repeating it.

## `gpu_mark_begin` / `gpu_mark_end`

**Contract** — push and pop a named marker into the device's command stream for an external frame-capture tool. Debug-only in effect, harmless in a shipping build; corresponds to the profiler seam.
