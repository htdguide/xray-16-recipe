# src/Layers/xrRender_R2/r2_rendertarget.cpp

> Builds the entire render-target set and every pass description the deferred path uses,
> and holds the small utilities — texture-coordinate generators, the light-marker counter,
> the light-shaft decision — that several passes share.

**Needs** — [`r2.h`](r2.h.md) · [`r2_types.h`](r2_types.h.md) ·
[`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) ·
[`xrRender/blenders/blender_light_occq.h`](../xrRender/blenders/blender_light_occq.h.md) ·
[`xrRender/blenders/blender_light_mask.h`](../xrRender/blenders/blender_light_mask.h.md) ·
[`xrRender/blenders/blender_light_direct.h`](../xrRender/blenders/blender_light_direct.h.md) ·
[`xrRender/blenders/blender_light_point.h`](../xrRender/blenders/blender_light_point.h.md) ·
[`xrRender/blenders/blender_light_spot.h`](../xrRender/blenders/blender_light_spot.h.md) ·
[`xrRender/blenders/blender_light_reflected.h`](../xrRender/blenders/blender_light_reflected.h.md) ·
[`xrRender/blenders/blender_combine.h`](../xrRender/blenders/blender_combine.h.md) ·
[`xrRender/blenders/blender_bloom_build.h`](../xrRender/blenders/blender_bloom_build.h.md) ·
[`xrRender/blenders/blender_luminance.h`](../xrRender/blenders/blender_luminance.h.md) ·
[`xrRender/blenders/blender_ssao.h`](../xrRender/blenders/blender_ssao.h.md) ·
[`xrRender/blenders/dx11MSAABlender.h`](../xrRender/blenders/dx11MSAABlender.h.md) ·
[`xrRender/blenders/dx11RainBlender.h`](../xrRender/blenders/dx11RainBlender.h.md) ·
[`xrRender/blenders/dx11MinMaxSMBlender.h`](../xrRender/blenders/dx11MinMaxSMBlender.h.md) ·
[`xrRender/ColorMapManager.h`](../xrRender/ColorMapManager.h.md) ·
[Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r4_rendertarget.h`](../xrRenderPC_R4/r4_rendertarget.h.md) · [`r4_rendertarget_phase_combine.cpp`](../xrRenderPC_R4/r4_rendertarget_phase_combine.cpp.md) · [`r2_rendertarget_accum_omnipart_geom.cpp`](r2_rendertarget_accum_omnipart_geom.cpp.md) · [`r2_rendertarget_accum_point_geom.cpp`](r2_rendertarget_accum_point_geom.cpp.md) · [`r2_rendertarget_accum_reflected.cpp`](r2_rendertarget_accum_reflected.cpp.md) · [`r2_rendertarget_accum_spot_geom.cpp`](r2_rendertarget_accum_spot_geom.cpp.md) · [`r2_rendertarget_draw_volume.cpp`](r2_rendertarget_draw_volume.cpp.md) · [`r2_rendertarget_enable_scissor.cpp`](r2_rendertarget_enable_scissor.cpp.md) · [`r2_rendertarget_phase_PP.cpp`](r2_rendertarget_phase_PP.cpp.md) · [`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md) · [`r2_rendertarget_phase_bloom.cpp`](r2_rendertarget_phase_bloom.cpp.md) · [`r2_rendertarget_phase_luminance.cpp`](r2_rendertarget_phase_luminance.cpp.md) · [`r2_rendertarget_phase_smap_S.cpp`](r2_rendertarget_phase_smap_S.cpp.md) · [`r3_rendertarget_accum_point.cpp`](r3_rendertarget_accum_point.cpp.md) · _and 8 more_
**Tier floor** — T1: it names device formats and multisample counts directly and decides
layouts by bit depth.

## Purpose

One long constructor and a short tail of shared helpers. The constructor is where the
G-buffer's shape is actually decided — the branching over what the device can do, in
[`r2.cpp`](r2.cpp.md), lands here as concrete formats. It also creates every pass
description the chapter's passes later select an element of, and every piece of fixed
geometry (the full-screen quads' vertex formats, the light volumes) they draw.

The render-target *object* is declared per backend (chapters 20 and 21) because its
handle types differ; its behaviour is this chapter's, and this file plus the
`phase_*`/`accum_*` files are that behaviour.

## State

The full target set, listed in [the chapter README](README.md#the-render-target-set-and-its-lifetime).
Beyond the targets, this object owns:

```text
RECORD RenderTargets
  width, height       : per command context      # the size each context renders at
  light_marker        : int      # the stencil value the current light is claiming
  accumulator_clear_frame : int  # which frame the accumulator was last cleared on
  bloom_factor        : real     # smoothed bloom intensity
  luminance_adapt     : real     # smoothed exposure adaptation rate
  noise_time, noise_shift_w, noise_shift_h : the film-grain animation state
  post parameters     : blur, gray, duality (h,v), noise (amount, scale, rate),
                        base colour, gray colour, additive colour,
                        colour-map influence and interpolation
  has_active_volumetric : bool   # whether the volumetric target has been touched
```

**Invariants** — the accumulator's clear-frame marker is what makes "bind the accumulator"
idempotent within a frame and clearing across frames; see
[`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md). The
light marker starts at five and advances by two, so it is always odd and always above the
values the G-buffer and emissive passes write.

## Construction — the G-buffer decision

**Contract** — creates every target and description. Runs once per device and again after
every resolution change. Fails only if the device rejects a format the option resolution
already promised.

```text
FUNCTION build_targets()
  sample_count  = multisampling ? sample setting : 1
  bound_samples = per-sample-on-demand ? 1 : sample setting
      # how many per-sample program variants must be compiled: one if the device can
      # select the sample at run time, otherwise one per sample

  # --- swap chain and depth ---
  one colour target per back buffer, at screen size, in the device's target format
  one depth target in the device's depth format
  multisample depth: a real multisampled target, or an alias of the above when off
      # aliasing rather than branching is what keeps the pass code free of MSAA tests

  # --- the G-buffer ---
  position: four 16-bit floats, at sample_count
  normal:   four 16-bit floats, at sample_count       # omitted when packed
  IF the device allows targets of differing bit depth
      albedo: four 8-bit         accumulator: four 16-bit floats
  ELSE IF the device can blend into 16-bit float targets
      albedo: four 16-bit floats (or 8-bit when packed)
      accumulator: four 16-bit floats
  ELSE                                     # weakest path
      albedo: four 8-bit
      accumulator: four 16-bit floats, plus a second one to blend through

  # --- scratch ---
  generic 0 and 1: four 8-bit, single-sampled; plus multisampled twins, or aliases
  generic (the third): four 8-bit
  generic 2 (volumetric): four 16-bit floats, only when advanced post is on
```

**Notes.** The three-way branch is the whole compatibility story of the era compressed
into one place. Mixed-depth targets let albedo stay eight bits while the accumulator is
float, which is the cheap and correct combination. Without that, but with float blending,
albedo is *promoted* to float so all three targets match — wasteful but working. Without
either, albedo stays eight bits and the accumulator cannot be blended into, so the
G-buffer pass borrows the accumulator as its albedo target and a full-screen pass copies
it across at the end of the fill (the *albedo work-around*, in
[`r3_rendertarget_phase_scene.cpp`](r3_rendertarget_phase_scene.cpp.md)), and every
accumulation ends with a blend-copy through the second accumulator. A rebuild targeting
modern hardware keeps only the first branch; it should still read the other two, because
the blend-copy tails are visible in half the accumulation code and would otherwise look
unmotivated.

The multisample aliasing is the other idea worth carrying: when multisampling is off, the
multisampled variants of the depth and the two scratch targets are made to *be* the
non-multisampled ones rather than being separate. Every pass then binds the same names
unconditionally.

## Construction — shadows

```text
  # spot atlas
  IF the device cannot compare-sample a depth texture
      create a colour surface as the atlas, and render linear depth into it
  ELSE
      the atlas is a depth texture in the vendor-preferred format, and the colour
      surface is either omitted or a null target
  the atlas is square, of the configured side
  it has one slice per sun cascade when the device supports target arrays, else one
  the rain map is its own square, at its own configured side
  IF min/max shadows are enabled
      a quarter-side single-channel float target, and the reduction description
```

**Notes** — the cascade array is a real decision, not an optimization detail: with array
targets all three cascades render into one texture and are accumulated together after the
last one, so the accumulator is bound once. Without them each cascade must be accumulated
immediately after it renders, because the next cascade will overwrite the texture. That
branch appears in [`render_phase_sun.cpp`](render_phase_sun.cpp.md) and is the only place
the two paths differ.

## Construction — pass descriptions

**Contract** — creates one description per lighting shape and per post-process stage, plus
a per-sample variant of each wherever multisampling needs one. Every variant is the same
description compiled with the sample index as a preprocessor value.

The set: occlusion query; stencil mask (one description, six elements — spot, point,
direct, volumetric accumulation, 2D accumulation, albedo resolve); direct (sun)
accumulation, in a cascade or a legacy flavour chosen by the same shader-source inspection
that chose the sun; direct volumetric, plus a min/max variant; rain; multisample edge
marking; point, spot, reflected and volumetric accumulation; ambient occlusion, in a
blurred or a compute-shader high-definition flavour; bloom build and filter; luminance
reduction; combine, plus a volumetric combine; the film post-process; the menu composite;
and several debug blits of a named target.

**Notes** — the light-shaft descriptions have their shadow-atlas texture *assigned by
hand* after creation, because the shipped program names a sampler that the material file
does not bind to any target. The engine looks the sampler up by name in the compiled
program's constant table and points it at whichever of the two atlas targets is real on
this device. That is a patch over a data/program mismatch in the shipped content; a
rebuild that ships its own material files should bind it in data and delete the patch.

Per-sample variants are compiled eagerly for every sample index when the device cannot
select a sample at run time, and once otherwise. That is why the sample count appears as
an array bound throughout the chapter: it is a *compile-time fan-out*, not a loop over
samples at draw time — although the draw code does also loop, setting a one-bit sample
mask per iteration, on the devices that need it.

## Construction — geometry

Full-screen work needs several vertex formats, and they are not interchangeable: a plain
transformed-and-lit quad for most blits; a two-coordinate-set quad for the reductions; a
four-coordinate-set quad for the bloom build (four taps per vertex); an eight-coordinate
quad for the separable filter (sixteen taps encoded as eight pairs); a position-only
declaration for the volumetric cuboid; and the three light volumes, built by
[`r2_rendertarget_accum_point_geom.cpp`](r2_rendertarget_accum_point_geom.cpp.md),
[`r2_rendertarget_accum_spot_geom.cpp`](r2_rendertarget_accum_spot_geom.cpp.md) and
[`r2_rendertarget_accum_omnipart_geom.cpp`](r2_rendertarget_accum_omnipart_geom.cpp.md).

The exposure pool is created here and *cleared to a mid value* rather than to zero, so the
first frame after a device reset is exposed reasonably instead of blinding white.

## `compute_screen_texgen`

**Contract** — produces the matrix that turns a light volume's clip-space position into a
texture coordinate for sampling the G-buffer. Pure.

```text
FUNCTION compute_screen_texgen(world_view_projection) -> matrix
  RETURN texel_adjust * world_view_projection
  # texel_adjust halves and biases x and y into [0,1]; it flips y on one backend,
  # because the two backends disagree about which corner is the texture origin
```

## `compute_jitter_texgen`

**Contract** — the same, then scaled so the jitter texture tiles across the screen at one
texel per pixel. Used by the shadow filter and by ambient occlusion to rotate their sample
kernel per pixel.

## `compute_noise_coordinates`

**Contract** — advances the film-grain animation and returns the coordinate rectangle for
this frame's noise. Reads the currently bound noise texture to learn its size, so the
material — not the engine — decides the grain's resolution.

```text
FUNCTION compute_noise_coordinates() -> (top_left, bottom_right)
  look up the noise sampler in the active program; FAIL if it is unbound
  tile = the bound texture's size, scaled by the grain setting
  noise_time = noise_time - frame_delta
  IF noise_time < 0
      pick a new random shift within the tile, on both axes
      add whole frame periods to noise_time until it is positive
  origin = (shift + half a texel) / tile
  RETURN origin, origin + screen size / tile
```

**Notes** — the shift is re-rolled on a *fixed* schedule, not every frame: film grain at
the frame rate looks like static, so it is re-rolled at a configured rate (the default is
well below the frame rate) and held between rolls. Accumulating whole periods rather than
resetting the timer keeps the rate honest when a frame is long.

## `compute_duality_coordinates`

**Contract** — returns two coordinate rectangles, one per eye of the "duality" effect: the
screen sampled twice with a horizontal and vertical offset, blended, which is how the game
renders drunkenness and concussion. Also folds in the blur amount as a half-texel
displacement. When the target's height differs from the screen's — a scaled render — the
blur is forced on, because point-sampling a scaled image aliases.

## `needs_post_process` / `needs_colour_map`

**Contract** — report whether any film post-process parameter is far enough from neutral
to be worth a pass. Neutral means: no blur, no desaturation, no grain, no duality, a base
colour within two of mid-grey on every channel, an additive colour within two of zero, and
no colour-map influence. The two-out-of-255 tolerance exists because the game layer
animates these parameters continuously and they linger microscopically off neutral.

## `reset_light_marker` / `increment_light_marker`

**Contract** — manage the stencil counter that identifies the light currently being
accumulated. Resetting optionally issues a full-screen pass that writes the stencil back
to its base value.

```text
FUNCTION reset_light_marker(also_clear_stencil)
  light_marker = 5
  IF also_clear_stencil
      draw a full-screen quad with the occlusion description's reset element

FUNCTION increment_light_marker()
  light_marker = light_marker + 2
  limit = multisampling ? 127 : 255      # the high bit is the edge flag
  IF light_marker > limit THEN reset_light_marker(clear = true)
```

**Invariants** — the marker is always odd, which is what lets the volume-marking passes
compare "at least one" against the low bit and "exactly this light" against the whole
value in the same buffer. It starts at five rather than one to leave room below for the
values the G-buffer pass (one) and the emissive pass write.

**Notes** — this counter is the reason a frame can carry two hundred lights without a
stencil clear between them. The clear only happens when the counter saturates, which at
two per light means once every hundred-odd lights — and under multisampling once every
sixty, since the top bit is reserved for the edge mask.

## `need_light_shafts` / `use_minmax_this_frame`

**Contract** — two coupled predicates. Light shafts are drawn when advanced post is on,
the user enabled them, and the current weather's shaft intensity is above a threshold —
the weather can switch them off entirely at midday or indoors. The min/max pyramid is
built when the option says always, or when the option says "follow the shafts" and the
shafts are on, or when the option says "decide by resolution" and the screen area is above
the vendor threshold *and* the shafts are on.

**Notes** — the intensity threshold is one ten-thousandth, which is "effectively zero"
rather than a tuning value; the comment in the source notes that the sun's colour should
also be folded in and is not. That is a real gap: a black sun with a non-zero shaft
intensity still pays for the shaft pass.

## Normal packing helpers

**Contract** — pack a unit vector into three bytes and unpack it again, used when a normal
must survive an eight-bit channel. Packing is a *search*: the direct quantization is
computed, then a small neighbourhood around it is scanned for the triple whose unpacked
direction has magnitude near one and the largest dot product with the original.

**Notes** — the search exists because naive per-channel rounding of a unit vector does not
in general unpack to a unit vector, and the error shows as shading bands on curved
surfaces. Scanning a radius of three in each axis — 343 candidates — is affordable because
this runs at build or load time, never per pixel. In debug builds the radius is zero,
which makes the function fast and slightly wrong, a trade only acceptable because the
result is visual.

These are vestigial on the current path: the G-buffer's normal is a float target, so
nothing calls them in the shipped configuration. They are the eight-bit fallback's
machinery.
