# src/Layers/xrRenderGL/glState.cpp

> Turns the engine's abstract state block into this API's two very different halves: a bag of global switches that must be set one at a time, and a per-stage sampler object that really is a unit.

**Needs** — [`glState.h`](glState.h.md) · [`glStateUtils.h`](glStateUtils.h.md) · [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) · [`xrRender/SH_Texture.h`](../xrRender/SH_Texture.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Blender_Recorder_GL.cpp`](Blender_Recorder_GL.cpp.md) · [`glState.h`](glState.h.md)
**Tier floor** — T1: sampler objects are device resources with explicit creation and destruction, and the block is applied by a burst of individual device calls on the frame's critical path.

## Purpose

This is the clearest place in the whole chapter to see what it costs to fill a seam shaped by another API. The interface demands "explicit rasterizer, blend, depth-stencil and sampler state objects, set as units". Of those four, this API genuinely has one — the sampler object. The other three do not exist: depth, stencil, blend and cull are global switches, each set by its own call.

So the block is a *record* rather than a device object, and applying it means replaying its fields. The mitigation is that most of the replay goes through the backend's redundancy cache ([`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md)), which drops a setting that already holds. The fields that do *not* go through the cache — the blend equation and factors, and the depth write mask — are therefore set unconditionally on every pass change, which is a real and avoidable cost a rebuild should notice.

## State

```text
RECORD StateBlock
  cull_mode          : ComparisonFunc-style enumerant (front / back / none)
  depth              : DepthStencilState
  blend              : BlendState
  applied_lod_bias   : real     # the mip bias this block's samplers were last
                                # written with; starts at +infinity so the first
                                # apply always writes
  samplers           : list<int> indexed by texture stage, length = the combined
                                 stage count; 0 means "no sampler object yet"

RECORD DepthStencilState
  depth_enable, depth_write : bool
  depth_func                : ComparisonFunc
  stencil_enable            : bool
  stencil_read_mask, stencil_write_mask : int (32-bit)
  stencil_fail_op, stencil_depth_fail_op, stencil_pass_op : StencilOp
  stencil_func              : ComparisonFunc
  stencil_ref               : int

RECORD BlendState
  enable                        : bool
  src_factor, dst_factor        : BlendFactor
  src_alpha_factor, dst_alpha_factor : BlendFactor
  op, alpha_op                  : BlendOp
  colour_mask                   : int (4 bits, one per channel)

# Invariant: a sampler object is created lazily, on the first sampler setting
# written to that stage, and every object the block created is destroyed when
# the block is. A stage with no object is left unbound at apply time, which
# means the device keeps whatever sampler the previous pass bound there --
# correct only because every pass that reads a texture also records at least
# one sampler setting for its stage.
```

**Defaults** — a freshly created block is not empty; it carries the state a pass gets when the material says nothing: depth test on, write on, less-or-equal; stencil enabled with an always-pass predicate, keep on every outcome, both masks all ones, reference zero; blend enabled but arithmetically a no-op (source times one plus destination times zero, add); all four colour channels written; cull counter-clockwise faces. A rebuild must reproduce these, because the shipped materials rely on them: a pass that sets only a blend factor still expects depth testing.

## `update_render_state(name, value)`

**Contract** — folds one pipeline-state pair from the material compiler's stream into the block. The name is in the frozen Direct3D 9 numbering, since that is what the shipped material scripts and the shared compiler speak; the value is stored raw and translated only at apply time. Unknown names abort in a checked build.

**Notes** — Four names are accepted and deliberately ignored: fixed-function lighting, fixed-function fog, alpha-test enable and alpha-test reference. The first two have no equivalent on a programmable pipeline. The last two matter more than they look: *alpha testing is no longer a pipeline state on this backend, it is something the pixel program does*, so the shipped material's alpha-test request reaches the device as a shader macro rather than as state. A rebuild targeting any modern API makes the same move, and must remember that the reference value has to travel to the shader instead.

The four per-target colour-write masks all fold into one field. This API's core profile can mask per attachment, but the shared core never asks for different masks on different targets, so the last one written wins and nothing is lost.

## `update_sampler_state(stage, name, value)`

**Contract** — folds one sampler setting into the stage's sampler object, creating the object on first use. Out-of-range stages are ignored rather than faulted. Writes go straight to the device object, not into a record — this is the half of the state block that really is an object, so there is nothing to defer.

```text
FUNCTION update_sampler_state(stage, name, value) -> ()
  IF stage outside [0, combined_stage_count) THEN RETURN

  current_min_filter := nearest
  IF samplers[stage] == 0 THEN
      samplers[stage] := device.create_sampler()
  ELSE IF name IS min_filter OR mip_filter THEN
      # The two D3D9 names write DIFFERENT HALVES of one device enumerant, so
      # whichever arrives second must read back what the first one wrote.
      current_min_filter := device.read(samplers[stage], min_filter)

  SELECT name
    address_u / address_v / address_w ->
        set wrap mode on the matching axis, translated by glStateUtils
    border_colour ->
        set the border colour from the packed integer's four channels
    magnification_filter ->
        set magnification filter, translated by glStateUtils
    minification_filter ->
        set combined filter := translate(value, current_min_filter, mip = false)
    mip_filter ->
        set combined filter := translate(value, current_min_filter, mip = true)
    mip_lod_bias -> set bias
    max_mip_level -> set the coarsest level index
    max_anisotropy ->
        set only if the device offers anisotropic filtering at all
    comparison_enable ->
        switch the sampler between plain sampling and depth comparison
    comparison_func ->
        set the comparison predicate
    otherwise -> fault: unimplemented sampler state
```

**Invariants** — the minification and mip filters are not independent on this API: they are two fields packed into one enumerant. The read-back above is what keeps a material that sets them in either order from losing the earlier one. This is the single most easily-missed translation in the file, and a rebuild on an API with separate fields simply stores both.

**Notes** — The comparison-enable and comparison-func names are not from Direct3D 9 at all; they are engine-invented names the material compiler emits for shadow-map samplers, because D3D9 had no comparison sampler and the newer renderers need one. They ride the same channel as the D3D9 names. A rebuild defining its own state vocabulary should note that the shipped material scripts can request a comparison sampler (`comp_less` in the scripting surface — see [`glResourceManager_Scripting.cpp`](glResourceManager_Scripting.cpp.md)) and must keep that reachable.

## `apply`

**Contract** — installs the block. Binds each stage's sampler object; rewrites the mip-bias-related sampler parameters when the user's bias setting has changed since this block was last applied; then replays cull, depth, stencil, depth write, blend and colour mask. Cull, depth test, depth function, stencil and colour mask go through the backend's redundancy cache; depth write and the blend equation and factors do not.

```text
FUNCTION apply() -> ()
  FOR EACH stage IN 0 .. combined_stage_count
    IF samplers[stage] == 0 THEN CONTINUE
    device.bind_sampler(stage, samplers[stage])

    # The user can change texture mip bias from the console at any time. Rather
    # than track a global generation number, each block remembers the bias its
    # samplers were written with and rewrites them when it no longer matches.
    IF applied_lod_bias differs from user_mip_bias THEN
        set min LOD 0, max LOD unbounded, bias := user_mip_bias
  applied_lod_bias := user_mip_bias

  backend.set_cull_mode(cull_mode)
  backend.set_depth_test(depth.depth_enable)
  backend.set_depth_func(depth.depth_func)
  backend.set_stencil(depth.stencil_enable, depth.stencil_func, depth.stencil_ref,
                      depth.stencil_read_mask, depth.stencil_write_mask,
                      depth.stencil_fail_op, depth.stencil_pass_op,
                      depth.stencil_depth_fail_op)
  device.set_depth_write(depth.depth_write)

  IF blend.enable THEN device.enable_blend ELSE device.disable_blend
  device.set_blend_factors(translate each of the four factors)
  device.set_blend_equations(translate op and alpha_op)
  backend.set_colour_mask(blend.colour_mask)
```

**Notes** — Binding samplers one at a time is the obvious cost here; the device offers a multi-bind when an extension is present and this code does not use it. Equally, the sampler bind loop walks every stage of the combined range on every pass, not just the stages the pass uses. Both are known and marked in the original as work not done.

## `release`

**Contract** — destroys every sampler object the block created, clears the array, and frees the block. The block is owned by the material system's intern table, so this runs at level teardown, not per frame.
