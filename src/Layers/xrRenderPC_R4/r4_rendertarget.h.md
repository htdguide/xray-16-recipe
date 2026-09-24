# src/Layers/xrRenderPC_R4/r4_rendertarget.h

> Declares the whole render-target set and pass surface of the deferred frame graph as the Direct3D 11 filling sees it: which targets exist, at what precision, and which phase entry points this filling adds over the shared ones.

**Needs** — [`../xrRender_R2/r2_rendertarget.cpp`](../xrRender_R2/r2_rendertarget.cpp.md) · [`../xrRender_R2/r2_types.h`](../xrRender_R2/r2_types.h.md) · [`../xrRender/ColorMapManager.h`](../xrRender/ColorMapManager.h.md) · [`../xrRenderDX11/dx11SH_RT.cpp`](../xrRenderDX11/dx11SH_RT.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r4_rendertarget_accum_direct.cpp`](r4_rendertarget_accum_direct.cpp.md) · [`r4_rendertarget_build_textures.cpp`](r4_rendertarget_build_textures.cpp.md) · [`r4_rendertarget_phase_combine.cpp`](r4_rendertarget_phase_combine.cpp.md) · [`r4_rendertarget_phase_hdao.cpp`](r4_rendertarget_phase_hdao.cpp.md) · [`r4_rendertarget_u_set_rt.cpp`](r4_rendertarget_u_set_rt.cpp.md) · [`stdafx.h`](stdafx.h.md)
**Tier floor** — T1: it is a table of device allocations with their view objects, and the precision of each entry is a bandwidth decision.

## Purpose

The frame graph of chapter 18 is written once and compiled by both modern backends. This
header is the Direct3D 11 backend's copy of the **declaration** it compiles against: the
complete target set, the compiled material descriptions each phase selects, the cached
geometry each full-screen pass draws, and the phase entry points. The implementation of
most of it lives in [`../xrRender_R2/r2_rendertarget.cpp`](../xrRender_R2/r2_rendertarget.cpp.md)
and its siblings; only the handful of files in this directory are replaced or added here.

The header is substantive in spite of that split, because the target *set* — what exists,
how wide each entry is and which of them survive a frame — is the contract the filling
owes the frame graph, and it is stated nowhere else. A rebuilder filling the seam with a
different device reads this list and allocates it.

## State

The set is created when the device comes up and rebuilt whenever the resolution changes.
Nothing here survives a resolution change except the exposure pool's *values*, and nothing
survives a frame boundary except the exposure pool itself.

```text
RECORD RenderTargetSet
  # --- what the current pass is writing into, per command context ---
  bound_width, bound_height : list<int>   # one entry per command context;
                                          # set by every bind, read by every full-screen pass
  light_marker              : int (8-bit) # the stencil value the current light owns
  accumulator_clear_frame   : int         # the frame the accumulator was last cleared

  # --- the swap chain ---
  back_buffers  : list<render target>     # one per buffer in the chain
  back_depth    : depth target

  # --- the G-buffer, screen sized ---
  depth         : depth target            # the scene's depth, written by the G-buffer fill
  msaa_depth    : depth target            # multisampled twin; ALIASES `depth` when
                                          # multisampling is off, so no pass branches on it
  position      : 4 x 16-bit float        # eye-space x,y,z + material id
  normal        : 4 x 16-bit float        # eye-space x,y,z + hemispheric-ambient factor
  albedo        : 4 x 16-bit float        # r,g,b + gloss  (8-bit per channel on weak devices)

  # --- lighting ---
  accumulator      : 4 x 16-bit float     # r,g,b diffuse + specular scalar
  accumulator_temp : 4 x 16-bit float     # only allocated where the device cannot blend
                                          # into a float target; see accum_direct_blend

  # --- scratch, screen sized ---
  generic_0, generic_1 : 4 x 8-bit        # post-process ping-pong / distortion mask
  generic_2            : 4 x 8-bit        # the volumetric (light-shaft) sum
  generic              : 4 x 8-bit        # the resolved low-dynamic-range image
  generic_0_resolved, generic_1_resolved  # multisampled twins of generic_0/1; ALIAS the
                                          # originals when multisampling is off

  # --- exposure ---
  bloom_1, bloom_2 : 4 x 8-bit, quarter size
  luminance_64     : 4 x 16-bit float, 64 x 64
  luminance_8      : 4 x 16-bit float, 8 x 8
  luminance_pool   : list<1 x 32-bit float, 1 x 1>   # two per GPU
  luminance_src, luminance_dest : texture handles    # re-pointed at pool entries each frame

  # --- shadows ---
  shadow_surface     : colour target, one configurable square
  shadow_depth       : depth target, the same square
  shadow_rain        : its own square
  shadow_depth_minmax: a reduced pyramid of the cascade, for light shafts

  # --- procedurally built, immutable, see r4_rendertarget_build_textures ---
  material_lookup : 3D, 2 x 8-bit, 128 x 256 x 4
  jitter[0..3]    : 2D, 4 x 8-bit signed, 64 x 64
  jitter[4]       : 2D, 4 x 32-bit float, 64 x 64   # the horizon-based occlusion kernel
  jitter_mipped   : the first jitter table with a point-filtered mip chain

  screenshot_staging : a screen-sized, host-readable image
```

**Invariants**

- **The multisampled twins alias their originals when multisampling is off.** This is the
  reason no pass in the frame graph branches on whether multisampling is enabled when it
  *binds* a target — only when it decides how many times to draw. A rebuild that skips the
  aliasing pays for it with a conditional in every bind site.
- **`bound_width`/`bound_height` are per command context, not global.** Several visibility
  walks are in flight at once, each with its own command list, and each has its own idea of
  what size the target it is writing is. Every full-screen pass computes its quad and its
  kernel scales from these, so they must follow the command list and not the device.
- **The exposure pool is the only entry that carries state across frames.** Its two
  entries per GPU are swapped at the end of the combine, which is both the double buffer
  and the way the value is kept off the critical path on a multi-GPU machine.
- **`shadow_surface` exists only because some devices refuse a depth-only target.** Where
  the device accepts depth with no colour attachment, it is never written.

The luminance pool is sized `2 × number of GPUs` and indexed by `frame number mod number
of GPUs`, so on a two-GPU machine each device reads back the exposure it itself measured
two frames ago. That staggering is what stops an exposure read-back from serializing the
two devices.

## Exported units

The type is one class; its surface divides into five groups.

**Target binding** — `u_setrt` in three arities (two targets, three targets, and an
explicit width/height/raw-view form), implemented in
[`r4_rendertarget_u_set_rt.cpp`](r4_rendertarget_u_set_rt.cpp.md); `get_base_rt` and
`get_base_zb` name the current back buffer and its depth.

**Full-screen helpers** — `u_stencil_optimize` (a vendor stencil-recompression hint),
`u_compute_texgen_screen` and `u_compute_texgen_jitter` (the matrices that turn a clip
position into a screen or jitter texture coordinate), `u_calc_tc_noise` and
`u_calc_tc_duality_ss` (post-process coordinate generation), `u_need_PP` / `u_need_CM`
(whether the film post-process and colour grading have anything to do this frame),
`u_DBT_enable` / `u_DBT_disable` (the depth-bounds hint, inert on this backend).

**Phases**, in frame order — `phase_scene_prepare`, `phase_scene_begin`, `phase_scene_end`,
`phase_occq`, `phase_ssao`, **`phase_hdao`**, `phase_downsamp`, `phase_wallmarks`,
`phase_smap_direct`, `phase_smap_direct_tsh`, `phase_smap_spot_clear`, `phase_smap_spot`,
`phase_smap_spot_tsh`, `phase_accumulator`, `phase_vol_accumulator`, `phase_rain`,
`phase_bloom`, `phase_luminance`, **`phase_combine`**, **`phase_combine_volumetric`**,
`phase_pp`. The three in bold are replaced or added by this filling; the rest are shared.

**Light accumulation** — `accum_direct`, `accum_direct_cascade`, `accum_direct_f`,
`accum_direct_lum`, `accum_direct_blend`, `accum_direct_volumetric` (all in
[`r4_rendertarget_accum_direct.cpp`](r4_rendertarget_accum_direct.cpp.md)); `accum_point`,
`accum_spot`, `accum_reflected`, `accum_volumetric`; `draw_volume`, `enable_scissor`,
`enable_dbt_bounds`; `reset_light_marker` and `increment_light_marker`.

**Construction** — `build_textures` (the procedural lookup tables, in
[`r4_rendertarget_build_textures.cpp`](r4_rendertarget_build_textures.cpp.md)) and the four
`accum_*_geom_create`/`destroy` pairs that own the light-volume meshes and the light-shaft
slice stack.

## Notes

**The light-shaft slice count is fixed at 100.** A light shaft is integrated by drawing a
stack of camera-facing slices through the light volume and summing the shadow term at each.
One hundred is the whole quality/cost knob for that effect and it is not exposed to the
player; a rebuild may make it adaptive, but must keep it high enough that the slices do not
band.

**The material-description handles are arrays indexed by sample count.** Every lighting
material that can run under multisampling is compiled once per sample count from one to
eight, because the number of samples is a compile-time constant inside the program. That
array — not a runtime uniform — is how this filling parameterizes multisampling, and it is
the single largest multiplier on the shader cache. See
[`r4_shaders.cpp`](r4_shaders.cpp.md) for the macro set that produces the variants.

**Debug line, sphere and plane buffers are compiled out of a shipping build.** They exist
so a pass can leave a marker to be drawn at combine time, which is the only point in the
frame where world-space debug geometry can be drawn against the finished depth buffer.
