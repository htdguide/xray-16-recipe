# src/Layers/xrRenderPC_GL/gl_rendertarget.h

> The frame's entire working set: every intermediate surface the deferred renderer allocates, every material it draws them with, and the phase vocabulary that consumes them.

**Needs** — [`gl_rendertarget_build_textures.cpp`](gl_rendertarget_build_textures.cpp.md) · [`gl_rendertarget_u_set_rt.cpp`](gl_rendertarget_u_set_rt.cpp.md) · [`gl_rendertarget_accum_direct.cpp`](gl_rendertarget_accum_direct.cpp.md) · [`gl_rendertarget_phase_combine.cpp`](gl_rendertarget_phase_combine.cpp.md) · [`../xrRenderGL/glSH_RT.cpp`](../xrRenderGL/glSH_RT.cpp.md) · [`../xrRender/ColorMapManager.h`](../xrRender/ColorMapManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`gl_rendertarget_accum_direct.cpp`](gl_rendertarget_accum_direct.cpp.md) · [`gl_rendertarget_build_textures.cpp`](gl_rendertarget_build_textures.cpp.md) · [`gl_rendertarget_phase_combine.cpp`](gl_rendertarget_phase_combine.cpp.md) · [`gl_rendertarget_phase_flip.cpp`](gl_rendertarget_phase_flip.cpp.md) · [`gl_rendertarget_u_set_rt.cpp`](gl_rendertarget_u_set_rt.cpp.md) · [`stdafx.h`](stdafx.h.md)
**Tier floor** — T1: it is a declaration of device memory layout — every field is a sized surface or a compiled material.

## Purpose

This is the deferred renderer's memory map and its phase vocabulary in one declaration, and it is substantive despite being a header: the *set* of surfaces, their formats and their sizes is the design of the renderer. The Direct3D 11 backend declares a near-identical set; where this one differs, the difference is a capability this backend lacks.

Read it as three things: the surfaces (what memory a frame occupies), the materials and geometry (what is drawn to fill them), and the phase methods (the order the frame is assembled in). The implementations are split across the other files in this directory and across the shared deferred path in [`xrRender_R2`](../xrRender_R2/README.md); the ones this batch covers are linked above.

## State

### The surfaces

```text
# --- the swap set. One per back buffer, plus one shared depth surface.
base[]           : the presentable colour surfaces
base_depth       : the depth-stencil surface shared by every phase that
                   needs the camera's depth

# --- the geometry buffer, written in one pass over the visible scene
position         : 64-bit, eye-space position with the material index packed in
normal           : 64-bit, eye-space normal with the hemisphere term packed in
colour           : 64- or 32-bit, albedo with specular gloss packed in
depth            : the initial depth pass's result

# --- the multi-sampling shadow set. When multi-sampling is off, each of these
#     is an ALIAS of its non-suffixed twin, so the phases need no branch.
msaa_depth       : aliases base_depth when off
generic_0_resolved, generic_1_resolved : alias generic_0 and generic_1 when off

# --- lighting
accumulator      : 64-bit, the sum of every light's contribution
accumulator_temp : only allocated on a device without floating-point blending,
                   which must therefore accumulate by copy (see accum_direct_blend)

# --- post-processing scratch
generic          : a full-size intermediate
generic_0, generic_1 : 32-bit full-size intermediates
generic_2        : 32-bit, the volumetric-light accumulation

# --- bloom and exposure
bloom_1, bloom_2 : 32-bit at quarter dimensions
luminance_64     : 64-bit, 64×64, holding a log-average in every channel
luminance_8      : 64-bit, 8×8, the next reduction step
luminance_pool[] : 1×1 single-float, TWO PER REPORTED GPU -- see Notes
luminance_source, luminance_destination : texture records pointed at two of the
                   pool entries each frame, swapped at frame end

# --- shadow maps
shadow_surface   : 32-bit colour, for the translucent-shadow pass
shadow_depth     : the depth the sun's shadow test reads
shadow_rain      : the rain occlusion map
shadow_minmax    : the coarse per-tile depth bounds, when min/max maps are on

# --- procedurally built lookup textures (see build_textures)
material_lookup  : 3D, the lighting model table
noise[]          : the jitter set; the last one is floating-point
noise_mipped     : a mipped copy of the first jitter texture

# --- per-frame scalars carried across phases
light_marker_id      : the stencil value the current light writes
accumulator_clear_mark : generation counter for the accumulator's clear
has_active_volumetric  : whether any light asked for a volumetric pass
bloom_factor, luminance_adaptation : the exposure pipeline's running values
```

**Invariants** — the aliasing of the multi-sample set is the single most useful thing on this page. When multi-sampling is disabled, the `_resolved` surfaces and the multi-sample depth are *the same objects* as their plain twins, so every phase can write to `generic_0_resolved` unconditionally and the resolve step becomes a no-op. A rebuild that allocates them separately doubles this part of the working set for no benefit; a rebuild that branches instead doubles the phase code.

**Invariants** — the pass width and height are recorded *per command-list context*, not globally, because several contexts record in parallel on the other backend and each may be rendering to a differently-sized target (a shadow map versus the main view). On this backend there is one context, but the indexing must be preserved or the shared phase code will not compile against it.

**Notes** — the luminance pool holds two surfaces per *reported* GPU and the combine phase selects a pair by frame number ([`gl_rendertarget_phase_combine.cpp`](gl_rendertarget_phase_combine.cpp.md)). The reported count is a hard-coded two ([`glHWCaps.cpp`](../xrRenderGL/glHWCaps.cpp.md)), so in practice this spreads the exposure feedback loop over two frames. See that file for what is and is not recoverable about the number.

### The materials and geometry

```text
# One compiled material per lighting or post-processing job, plus a parallel
# multi-sampled variant array (eight entries: one per sample count) for each
# that participates in the multi-sampled path.
occlusion query · accumulator mask · direct (sun) · direct volumetric ·
direct volumetric min/max · point light · spot light · reflected light ·
volume light · min/max map generation · rain · multi-sample edge marking ·
ambient occlusion · bloom · luminance · combine · volumetric combine ·
post-process · menu

# Geometry the phases draw with. The light volumes are staged once and
# uploaded; the quads are filled from the per-frame stream each time.
point-light volume, spot-light volume, omni partition, volumetric slices
   -- each with its own staged vertex and index buffer
combine quad, combine quad with clip-space positions, two-coordinate quad,
   combine cuboid, blur quad, antialias quad, bloom build/filter quads,
   post-process quad, menu quad
```

**Invariants** — the multi-sample variants are indexed by sample count, and this backend supports only the *optimized* multi-sample configuration; the per-sample loops that the other backend runs abort here (see [`gl_rendertarget_accum_direct.cpp`](gl_rendertarget_accum_direct.cpp.md)). The arrays are still eight wide because the shared code indexes them that way.

**Notes** — `VOLUMETRIC_SLICES` is declared as one hundred, with a note that it must be at least two. It is the number of slices the volumetric light volume is built from, and it trades sun-shaft banding against fill rate. A rebuild should treat it as tunable.

### The post-processing parameters

```text
blur, grayscale, horizontal and vertical duality, noise amount, noise scale,
noise rate, base colour, gray colour, additive colour,
colour-map influence, colour-map interpolation, and a colour-map manager
```

These are the game-driven post-processing effects — the screen goes gray and blurs when the player is hurt, doubles when drugged, and so on. They are set by the game layer through the setters at the bottom of the declaration and consumed by the post-process phase.

## The phase vocabulary

**Contract** — the frame's assembly order, as a list of named phases. Each is implemented in this directory or in the shared deferred path.

```text
# --- setup
build_textures                     procedural lookup tables, once at startup
accum_*_geom_create / _destroy     the light volumes, once at startup

# --- target binding (this batch: gl_rendertarget_u_set_rt.cpp)
set_targets(...)                   four overloads; see that file
stencil_optimize(mode)             recompress the stencil buffer
compute_screen_texgen / jitter_texgen    the screen- and noise-space matrices
need_post_process / need_colour_map      whether the optional phases run
depth_bounds_enable / _disable     unimplemented on this backend

# --- the frame
scene_prepare, scene_begin, scene_end
occlusion_query
ambient_occlusion, downsample
wallmarks
shadow_map_direct(light, phase) and its translucent form
shadow_map_spot_clear, shadow_map_spot, and its translucent form
accumulator, volumetric_accumulator
shadow_direct(light, phase)
create_minmax_shadow_map
rain, draw_rain(light)
mark_multisample_edges
draw_volume(light)
accum_direct(phase) and its cascade, filtered, luminance,
    blend and volumetric forms          (this batch: accum_direct.cpp)
accum_point(light), accum_spot(light), accum_reflected(light),
    accum_volumetric(light)
bloom, luminance
combine, combine_volumetric          (this batch: phase_combine.cpp)
post_process
```

**Notes** — the presentation phase is declared but compiled out; see [`gl_rendertarget_phase_flip.cpp`](gl_rendertarget_phase_flip.cpp.md). Presentation is a framebuffer copy in [`glHW.cpp`](../xrRenderGL/glHW.cpp.md) instead.

## `base_render_target` · `base_depth_target`

**Contract** — the current back buffer's colour surface and the shared depth surface. The colour one indexes the swap set by the device's current back-buffer counter, which on this backend always yields the single entry.

## `light_marker` operations

**Contract** — the deferred lighting path marks each light's affected pixels in the stencil buffer with a per-light identifier and then shades only those. `increment_light_marker` advances the identifier; `reset_light_marker` returns it to the start and, when asked, clears the stencil. The reset takes a flag because **the stencil only needs clearing when the marker wraps**, and clearing a full-resolution stencil buffer per light would dominate the lighting cost.

## `dimensions(command_list)`

**Contract** — the current pass's target width and height, per context. Written by the target-binding calls; read by every phase that builds a full-screen quad.
