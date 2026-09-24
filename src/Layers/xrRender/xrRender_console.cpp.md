# src/Layers/xrRender/xrRender_console.cpp

> The renderer's entire tuning surface: every `r__`/`r1_`/`r2_`/`r3_` console name, what it controls, what it is clamped to, and what it starts at.

**Needs** — [`xrRender_console.h`](xrRender_console.h.md) · [`xrEngine/xr_ioc_cmd.h`](../../xrEngine/xr_ioc_cmd.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [`xrCore/xr_token.h`](../../xrCore/xr_token.h.md) · [`DetailManager.h`](DetailManager.h.md) · [`ModelPool.h`](ModelPool.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`r__pixel_calculator.h`](r__pixel_calculator.h.md) · [`xrCore/Animation/SkeletonMotions.hpp`](../../xrCore/Animation/SkeletonMotions.hpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)

**Used by** — [`xrRender_console.h`](xrRender_console.h.md)

**Tier floor** — T2: names, clamp ranges and storage bindings. Two of the variables reach the device when they change, and they do it through the backend's own sampler-state cache.

## Purpose

This file is a **catalogue, not an algorithm**. It defines the storage for every renderer
setting, gives each one a console name, a clamp range and a starting value, and registers
the handful of settings that need to do something when they change.

It matters far more than its shape suggests, because these names are a **frozen
user-facing surface**. The shipped quality presets, every player's settings file and a
decade of modifications address these variables by name with the argument syntax the
console defines (system requirements §6, criterion 3: the console must accept every
shipped variable name and write back a settings file the original also accepts). A rebuild
may implement any of them differently and may not rename one, may not narrow a clamp range
a shipped preset relies on, and may not change what an existing value means.

The variables are **module globals the render code reads directly** in its inner loops —
there is no getter and no change notification. Where a setting cannot be honoured
mid-frame, the answer is not a notification but a restart: a handful of names are marked
below as taking effect only on the next renderer bring-up, and that is a fact the options
screen also encodes.

The `r`-number in a name is historical: it says which renderer generation introduced the
setting, not which renderers read it. Several `r2_` names are read by the DirectX 11 and
OpenGL backends and by nothing called R2; `r1_detail_textures` toggles a bit that lives in
the R2 flag word. Treat the prefixes as opaque name parts.

## State

All storage in this file is process-global and lives for the renderer's life. Four of the
globals are **flag words** — a set of independent booleans packed into one 32-bit value,
each exposed under its own console name:

```text
RECORD RenderSettings              # module-global; read directly, never through an accessor
  common_flags  : int (32-bit)     # renderer-independent toggles
  r1_flags      : int (32-bit)     # fixed-function-era toggles
  ls_flags      : int (32-bit)     # the main lighting/shadowing/post toggle word
  ls_flags_ext  : int (32-bit)     # the overflow word, added when the first filled up
  <one scalar or token per row of the tables below>
```

**Invariants**

- A flag word is packed only to make it cheap to test and to save; **bit positions are
  internal** and a rebuild may use a set of names instead. What is frozen is the *console
  name* of each bit and the fact that it saves as `0` or `1`.
- `ls_flags_ext` exists purely because `ls_flags` ran out of bits. The split is arbitrary
  and a rebuild should merge them.
- Six bits of `ls_flags` have **no console name at all** (see the flag table). They are set
  once by the defaults and by backend bring-up and are then unreachable from the console —
  effectively compile-time decisions wearing a runtime costume.
- A token variable stores the *numeric* value behind the token, and the settings file
  stores the *token text*. Adding a token is safe; renaming or renumbering one silently
  changes what a shipped preset means.

## `xrRender_initconsole`

**Contract** — registers the renderer's whole command set with the console. Called once,
during renderer bring-up, after the engine has registered its own set. Allocates one
command object per name; those objects live until the process ends. Not re-entrant and
never called twice.

**Invariants**

- It runs **after** the engine's own registration, and the console's name table is
  last-writer-wins. Two names are registered twice as a result, and the renderer's
  registration is the one that survives: `r__supersample` (the renderer widens the engine's
  range from 1..4 to 1..8) and, in effect, `r1_tf_mipbias`/`r2_tf_mipbias`, which are two
  names bound to *one* variable so that a settings file written by either renderer restores
  the same value. The first of those is an accident and the second is deliberate; a rebuild
  should make the deliberate one explicit as an alias and drop the accident.
- Registration order is otherwise irrelevant; nothing here reads another variable at
  registration time.

In the tables below, **†** marks a name registered only in a developer build — it does not
exist in the shipping build and a settings file that mentions it is rejected there.

### Common — every renderer

| Name | Range | Default | Controls |
|---|---|---|---|
| `r__geometry_lod` | 0.1 .. 2 | 0.75 | Global scale on the screen-area thresholds at which a model drops a level of detail |
| `r__supersample` | 1 .. 8 | 1 | Render-resolution multiplier. **Nothing reads it** — see Notes |
| `r__detail_density` | 0.1 .. 0.99 | 0.6 | How much of the grass-and-debris layer is placed per unit area |
| `r__detail_radius` | 49 .. 300 | 49 | Radius of the detail-object layer, in world units. Derives the detail cache geometry — see its own section |
| `r__detail_height` | 1 .. 2 | 1 | Vertical scale on detail objects |
| `r__detail_l_ambient` † | 0.5 .. 0.95 | 0.9 | Ambient term used when lighting detail objects |
| `r__detail_l_aniso` † | 0.1 .. 0.5 | 0.25 | Directional term used when lighting detail objects |
| `r__dtex_range` | 5 .. 175 | 50 | Distance, in world units, over which detail textures fade out |
| `r__tf_aniso` | 1 .. 16 | 8 | Anisotropic filtering taps. Applies immediately — see its own section |
| `r1_tf_mipbias` / `r2_tf_mipbias` | -3 .. +3 | 0 | Mip-selection bias. One variable, two names. Applies immediately |
| `r__actor_shadow` | flag | on | Whether the player's own model casts a shadow |
| `r__wallmark_ttl` | 1 .. 600 | 50 | Seconds a decal survives before it is recycled |
| `r__wallmark_shift_pp` † | 0 .. 1 | 0.0001 | Offset a decal is pushed along the surface normal, to beat depth fighting |
| `r__wallmark_shift_v` † | 0 .. 1 | 0.0001 | The same offset applied to the decal's vertices |
| `r__lsleep_frames` † | 4 .. 30 | 10 | Frames a light may go unseen before it stops being updated |
| `r__ssa_glod_start` † | 128 .. 512 | 256 | Screen area at which a grouped level-of-detail crossfade begins |
| `r__ssa_glod_end` † | 16 .. 96 | 64 | Screen area at which it finishes |
| `r__clear_models_on_unload` | 0 .. 1 | 0 | Whether the model pool is emptied when a level unloads, trading a load stall for memory |
| `rs_skeleton_update` | 2 .. 128 | 32 | Frames between full skeleton re-evaluations for an off-screen or distant creature |

### R1 — the fixed-function-era renderer

| Name | Range | Default | Controls |
|---|---|---|---|
| `r1_ssa_lod_a` | 16 .. 96 | 64 | Screen area below which a model drops to its next level of detail |
| `r1_ssa_lod_b` | 16 .. 64 | 48 | Screen area below which it drops again |
| `r1_lmodel_lerp` | 0 .. 0.333 | 0.1 | Blend between the two lighting models used for static geometry |
| `r1_dlights` | flag | on | Whether dynamic lights are drawn at all |
| `r1_dlights_clip` | 10 .. 150 | 40 | Distance beyond which a dynamic light is dropped |
| `r1_glows_per_frame` | 2 .. 32 | 16 | Cap on glow sprites drawn in one frame |
| `r1_detail_textures` | flag | off | Whether the second, close-range texture layer is applied |
| `r1_fog_luminance` | 0.2 .. 5 | 1.1 | Brightness multiplier on fog |
| `r1_pps_u`, `r1_pps_v` | -1 .. +1 | 0 | Screen-space offsets applied to the post-process overlay |
| `r1_force_geomx` | 0 .. 1 | 0 | Force the extended vertex declaration path |
| `r1_software_skinning` | 0 .. 2 | 0 | 0 = renderer chooses, 1 = skin on the processor, 2 = force skinning on the device |
| `r1_ffp` | flag | off | Avoid shaders entirely: fixed-function or processor-side transform |
| `r1_ffp_lightmaps` | flag | off | The same for the lightmap pass |

### R2 and later — lighting, shadows and the sun

| Name | Range | Default | Controls |
|---|---|---|---|
| `r2_ssa_lod_a` | 16 .. 96 | 64 | First level-of-detail screen-area threshold |
| `r2_ssa_lod_b` | 32 .. 64 | 48 | Second threshold |
| `r2_sun` | flag | on | Whether the sun casts shadows |
| `r2_sun_details` | flag | off | Whether detail objects appear in the sun's shadow map |
| `r2_sun_focus` | flag | on | Fit the sun's projection to what the camera can actually see |
| `r2_sun_tsm` | flag | on | Use the trapezoidal shadow-map projection |
| `r2_sun_tsm_proj` | 0.001 .. 0.8 | 0.3 | Strength of that projection's warp |
| `r2_sun_tsm_bias` | -0.5 .. +0.5 | -0.2 | Depth bias applied under it |
| `r2_sun_near` | 1 .. 150 | 20 | Distance covered by the near shadow cascade |
| `r2_sun_far` | 51 .. 180 | 100 | Distance covered by the far cascade |
| `r2_sun_near_border` | 0.5 .. 1 | 0.75 | Fraction of the near cascade before the blend to the next begins |
| `r2_sun_depth_near_scale` / `_far_scale` | 0.5 .. 1.5 | 1.0 | Per-cascade depth scale, against shadow acne |
| `r2_sun_depth_near_bias` / `_far_bias` | -0.5 .. +0.5 | +0.00001 / -0.00002 | Per-cascade depth bias, same purpose |
| `r2_sun_lumscale` | -1 .. +3 | 1 | Sun contribution to scene luminance. Negative is legal and darkens |
| `r2_sun_lumscale_hemi` | 0 .. +3 | 1 | Sky-dome contribution |
| `r2_sun_lumscale_amb` | 0 .. +3 | 1 | Ambient contribution |
| `r2_sun_quality` | token: low / medium / high / ultra / extreme | medium | Shadow-map sampling quality. The top two tokens exist only on the Direct3D backend |
| `r2_sun_shafts` | token: off / low / medium / high | medium | Volumetric shafts through the sun's shadow map |
| `r2_smap_size` | token: 1024 .. 16384 (and 256 / 512 in a developer build) | 2048 | Shadow-map edge length in pixels |
| `r2_ls_depth_scale` | 0.5 .. 1.5 | 1.00001 | Depth scale for local shadow-casting lights |
| `r2_ls_depth_bias` | -0.5 .. +0.5 | -0.0003 | Depth bias for the same |
| `r2_ls_squality` | 0.5 .. 1 | 1 | Local shadow-map resolution scale |
| `r2_ls_dsm_kernel` | 0.1 .. 3 | 0.7 | Filter width for a directional light's shadow |
| `r2_ls_psm_kernel` | 0.1 .. 3 | 0.7 | Filter width for a point light's |
| `r2_ls_ssm_kernel` | 0.1 .. 3 | 0.7 | Filter width for a spot light's |
| `r2_slight_fade` | 0.2 .. 1 | 0.5 | Distance falloff applied to small lights |
| `r2_allow_r1_lights` | flag | off | Let the old renderer's unshadowed lights run alongside the deferred ones |
| `r2_zfill` | flag | off | Run a depth-only prepass before shading |
| `r2_zfill_depth` | 0.001 .. 0.5 | 0.25 | Fraction of the view distance that prepass covers |
| `r2_gi` | flag | off | The photon-walk global-illumination experiment |
| `r2_gi_depth` | 1 .. 5 | 1 | Bounces walked |
| `r2_gi_photons` | 8 .. 256 | 16 | Photons per bounce |
| `r2_gi_clip` | ~0 .. 0.1 | ~0 | Contribution below which a photon is dropped |
| `r2_gi_refl` | ~0 .. 0.99 | 0.9 | Energy retained per bounce |
| `r2_dhemi_count` † | 4 .. 25 | 5 | Samples taken when measuring a point's sky visibility |
| `r2_dhemi_sky_scale` † | 0 .. 100 | 0.08 | Weight of the sky in that measurement |
| `r2_dhemi_light_scale` † | 0 .. 100 | 0.2 | Weight of local lights in it |
| `r2_dhemi_light_flow` † | 0 .. 1 | 0.1 | Rate the measurement is allowed to change |
| `r2_dhemi_smooth` † | 0 .. 10 | 1 | Temporal smoothing on the result |
| `r2_shadow_cascede_zcul` † | flag | off | Depth-cull each shadow cascade against the previous (the name's misspelling is shipped) |
| `r2_shadow_cascede_old` † | flag | off | Use the pre-cascade sun shadow path |
| `rs_hom_depth_draw` † | flag | off | Draw the software occlusion map's depth buffer over the frame |
| `r2_use_nvdbt` † | flag | off | Use a vendor depth-bounds extension |
| `r2_mt` † | flag | on | Run visibility and light calculation on a worker |
| `r2_mt_calculate` | 0 .. 1 | 1 | The same toggle, available in the shipping build |
| `r2_mt_render` | 0 .. 1 | 1 | Record draw commands on a worker. Direct3D 11 only |
| `r2_wait_sleep` | 0 .. 1 | 0 | Whether a worker sleeps or spins while waiting |
| `r2_wait_timeout` | 100 .. 1000 | 500 | Milliseconds before a worker wait is abandoned |

### R2 and later — surfaces and post-processing

| Name | Range | Default | Controls |
|---|---|---|---|
| `r2_tonemap` | flag | on | Map high-range luminance to the display |
| `r2_tonemap_middlegray` | 0 .. 2 | 1 | The luminance that maps to mid-grey |
| `r2_tonemap_adaptation` | 0.01 .. 10 | 1 | How fast the eye adapts to a change |
| `r2_tonemap_lowlum` | 0.0001 .. 1 | 0.0001 | Floor on measured scene luminance, so a black frame does not blow out |
| `r2_tonemap_amount` | 0 .. 1 | 0.7 | Blend between the tone-mapped and the raw image |
| `r2_ls_bloom_threshold` | 0 .. 1 | 0.00001 | Luminance above which a pixel blooms |
| `r2_ls_bloom_speed` | 0 .. 100 | 100 | How fast the bloom's luminance measurement tracks |
| `r2_ls_bloom_kernel_scale` | 0.5 .. 2 | 0.7 | Overall bloom radius |
| `r2_ls_bloom_kernel_g` | 1 .. 7 | 3 | Width of the gaussian blur pass |
| `r2_ls_bloom_kernel_b` | 0.01 .. 1 | 0.7 | Width of the cheaper bilinear pass |
| `r2_ls_bloom_fast` | flag | off | Use the bilinear pass instead of the gaussian |
| `r2_aa` | flag | off | Edge-detecting post-process antialiasing |
| `r2_aa_kernel` | 0.3 .. 0.7 | 0.5 | Its sampling radius |
| `r2_aa_break` | vector in (0,0,0) .. (1,1,1) | (0.8, 0.1, 0) | Depth, normal and (unused) thresholds at which an edge is declared |
| `r2_aa_weight` | vector in (0,0,0) .. (1,1,1) | (0.25, 0.25, 0) | Weights the two detected edge kinds are blended with |
| `r2_mblur` | 0 .. 1 | 0 | Motion blur strength. Off by default |
| `r2_dof_enable` | flag | on | Depth of field |
| `r2_dof` | vector, ordered | (-1.25, 1.4, 600) | Near, focus and far distances at once — see its own section |
| `r2_dof_near` / `r2_dof_focus` / `r2_dof_far` | -10000 .. 10000, ordered | as above | The same three, one at a time |
| `r2_dof_kernel` | 0 .. 10 | 5 | Blur radius at full defocus |
| `r2_dof_sky` | -10000 .. 10000 | 30 | Distance the sky is treated as being at |
| `r2_ssao` | token: off / low / medium / high / ultra | high | Ambient-occlusion sample count |
| `r2_ssao_mode` | token: disabled / default / hdao / hbao | default | Which occlusion algorithm — see its own section |
| `r2_ssao_blur`, `r2_ssao_opt_data`, `r2_ssao_half_data`, `r2_ssao_hbao`, `r2_ssao_hdao` | flag | half_data on, rest off | The individual occlusion bits `r2_ssao_mode` drives. **Restart** |
| `r2_parallax_h` | 0 .. 0.5 | 0.02 | Apparent height of a parallax-mapped surface |
| `r2_steep_parallax` | flag | on | Use the ray-marched parallax instead of the offset approximation |
| `r2_detail_bump` | flag | on | Apply the close-range normal-map layer |
| `r2_soft_water` | flag | on | Fade water against the geometry behind it. **Restart** |
| `r2_soft_particles` | flag | on | The same for particles. **Restart** |
| `r2_volumetric_lights` | flag | on | Light shafts from local lights |
| `r2_gloss_factor` | 0 .. 10 | 4 | Global multiplier on material gloss |
| `r2em` | 0 .. 4, or `on` / `off` | off | Override every material with one lighting model — see its own section |
| `r3_water_refl` | token: off / low / medium / high / ultra | high | Screen-space reflection quality on water |
| `r3_water_refl_half_depth` | flag | on | Trace those reflections against a half-resolution depth buffer |
| `r3_water_refl_jitter` | flag | on | Jitter the ray start, trading banding for noise |
| `r3_dynamic_wet_surfaces` | flag | on | Wet the world under rain |
| `r3_dynamic_wet_surfaces_near` | 5 .. 70 | 5 | Near edge of the wetness map |
| `r3_dynamic_wet_surfaces_far` | 20 .. 100 | 20 | Far edge |
| `r3_dynamic_wet_surfaces_sm_res` | 64 .. 2048 | 256 | Resolution of the map that decides what the rain reaches |
| `r3_volumetric_smoke` | flag | on | The simulated smoke volume |
| `r3_msaa` | token: off / 2x / 4x / 8x | off | Multisample count |
| `r3_msaa_alphatest` | token: off / dx10_0 / dx10_1 | off | How alpha-tested surfaces are resolved under multisampling |
| `r3_gbuffer_opt` | flag | on | The compact geometry-buffer layout |
| `r3_use_dx10_1` | flag | off | Use the feature level's extra capabilities where present |
| `r3_minmax_sm` | token: off / on / auto / autodetect | autodetect | Use a min/max shadow-map hierarchy to skip filtering taps |
| `r4_enable_tessellation` | flag | on | Tessellate where a material asks for it. **Restart** |
| `r4_wireframe` | flag | off | Draw everything as wireframe. **Restart** |

### Flag bits with no console name

Six bits of the main flag word are set by the defaults or by backend bring-up and cannot
be reached from the console: split the scene into shadowed and unshadowed passes; skip the
visibility test for unshadowed lights; use a vendor stencil extension; ignore portals when
building the sun's shadow map (default off); and two multisampling variants (hybrid,
optimized — both off). They are on by default except where noted. A rebuild should decide
each one at build time and delete the bit, or expose it; the current state is the worst of
both.

### Commands

| Name | Effect |
|---|---|
| `screenshot [name]` | Capture the frame. Does nothing on a dedicated server |
| `_preset <token>` | Load a whole quality preset — see its own section |
| `render_memory_stats` | Print device memory by resource class. Implemented on the Direct3D backend only |
| `dump_resources` † | Print the model pool and the shader/texture resource tables |
| `stat_models` † | Print the model pool alone |
| `stat_textures` † | Print texture memory use |
| `stat_motions` † | Print the shared animation bank's contents |
| `build_ssa` † | Run the offline screen-area calibration pass. Not on the oldest renderer |
| `r3_fog_reload` † | Re-read the volumetric fog profiles from configuration, for live tuning |

**Notes** — `r__supersample` is registered, clamped, saved and restored, and **nothing
anywhere reads it**. It is registered twice, with two different upper bounds, which is how
it survived: the duplicate hid the fact that the variable was orphaned. A rebuild must
still accept and round-trip the name, because shipped settings files contain it, and need
not implement anything behind it.

Two more globals are defined here and read nowhere: a steep-parallax mode selector and an
"extended quality" token list. They are dead and carry no console name.

## `_preset`

**Contract** — a token variable over five quality levels (minimum, low, default, high,
extreme; default is the middle one). Setting it does not itself change any renderer
setting: it maps the token to a configuration file name, resolves that name against the
game's configuration root, and asks the console to execute its file-loading command on it.

```text
FUNCTION apply_preset(level)
  file = "rspec_" + name_of(level) + ".ltx"    # minimum|low|default|high|extreme
  path = resolve file against the game configuration root
  console.execute("cfg_load " + path)          # each line of which is a console command
```

**Invariants** — the preset files ship with the game and are plain lists of console
commands, so **a preset can set anything the console can set**, including variables this
file does not own. That is the whole design: presets are data, the engine holds no table
of what a preset means, and a modification retunes the game by editing five text files.

The stored value is only the token last chosen. Nothing re-reads the preset file, and
nothing detects that the player has since changed an individual setting — so the recorded
preset and the actual configuration drift apart immediately and legitimately.

## `r2_ssao_mode`

**Contract** — a token variable that is also a **macro over five other variables**.
Choosing a mode sets the occlusion sample-count variable and four flag bits to a
self-consistent combination, because the five underlying switches have invalid
combinations (two algorithms at once, an algorithm enabled with zero samples) that the
options screen must not be able to produce.

```text
FUNCTION apply_ssao_mode(mode)
  IF mode = disabled
    sample_quality = off;  clear the two algorithm bits;  RETURN

  IF sample_quality = off  THEN sample_quality = lowest   # never leave it enabled at zero

  IF mode = classic        clear both algorithm bits;  clear half-resolution
  IF mode = horizon_based  set horizon bit;  clear the other;  set packed-data
  IF mode = high_definition set high-definition bit;  clear the other
                           clear packed-data;  clear half-resolution
```

**Notes** — The individual bits remain separately settable by name, so the console can
still reach a combination the mode selector would never produce. That is deliberate —
they were the original interface and settings files contain them — and it means the mode
variable is a *convenience that can go stale*, not the authority. A rebuild should keep
the individual names working and treat the mode as a setter only.

## `r__detail_radius`

**Contract** — an integer radius in world units that, on every change, recomputes the four
derived quantities describing the detail-object cache: its half-extent in cache cells, the
cell count along one edge, the total cell count, and the distance at which detail objects
fade out.

```text
FUNCTION apply_detail_radius(radius)
  half_extent = floor(radius / 4) * 2      # rounded to an even number of cells
  stride      = half_extent * 2 / 4        # 4 = the cache's sub-block count
  edge        = half_extent + 1 + half_extent   # cells each side, plus the centre cell
  cell_count  = edge * edge
  fade        = 2 * half_extent - 0.5      # just inside the outermost ring
```

**Invariants** — the cache is a square grid centred on the camera with an odd edge length,
which is what the `+1` buys: there is exactly one centre cell, and the camera is always
inside it. The division by four is the sub-block count the detail cache is built from
([`DetailManager.h`](DetailManager.h.md)), and `half_extent * 2` must stay divisible by it —
which the rounding to an even cell count guarantees. The fade distance is set half a unit
inside the outermost ring so that an object never pops in at full opacity at the cache
boundary.

There are two copies of every one of these: a "current" set the detail manager is running
on, and a pending set this command writes. The renderer compares them each frame and
rebuilds the cache when they differ, because resizing the grid cannot happen mid-frame.

## `r__tf_aniso` and the mip-bias pair

**Contract** — texture filtering settings that must reach **already-created sampler state**,
not just the next-created one. Both re-assert themselves on the backend's sampler cache
whenever they are set *and whenever their value is printed*, and both no-op when there is
no device yet — which is exactly the case during startup, when the settings file is
replayed before the device exists.

**Notes** — Re-applying on a *read* is not defensive coding, it is the recovery path: a
device reset rebuilds every sampler from defaults and there is no notification, so the
options screen reading the value back is what restores it. That is a hack a rebuild should
replace with a device-reset hook, and the requirement it satisfies — sampler settings
survive a device reset — must survive with it.

The anisotropy value is clamped a second time, to 1..16, on the way to the device. The
console clamp already guarantees it; the second clamp is where the *hardware's* limit is
expressed, and a rebuild should clamp against the device's reported maximum instead of a
constant.

## `r2em` — the global material override

**Contract** — a float variable that also accepts the words `on` and `off`, which toggle a
flag bit rather than setting the number. When the flag is on, the number's integer part
selects one of four lighting models in a fixed order (oren-nayar, blinn, phong, metal) and
its fractional part blends into the next, wrapping around. Setting the number while the
flag is on prints the resulting pair and blend factor.

**Notes** — This is a developer instrument for comparing lighting models across a whole
level, not a player setting. Its constructor forces the stored value to zero, discarding
the initializer the variable was defined with — so the declared default is unreachable and
the override always starts at the first model. That looks like an accident and behaves
like one; a rebuild should pick one and delete the other.

## The depth-of-field quartet

**Contract** — one vector variable and three scalar ones over the same three numbers (near,
focus, far), all four enforcing the ordering `near ≤ focus - 0.1 ≤ far - 0.2`. A value that
would break the ordering is **rejected with an explanation and the conflicting variable's
current value is printed**, rather than being clamped into place. On success all four push
the whole vector to the game layer, which owns the camera's base depth of field and blends
the engine's value against whatever the current weapon or cutscene asks for.

**Invariants**

- The three scalar names **do not serialize**. Only the vector name writes itself to the
  settings file, so the three numbers round-trip exactly once and cannot be restored in an
  order that violates the constraint. This is the load-bearing part: a naive rebuild that
  saves all four produces a settings file that fails to load its own third line.
- The minimum-distance bound is not zero. The shipped default puts the near plane behind
  the camera, which is how "nothing in front of the player is defocused" is expressed.

## `xrRender_test_hw`

**Contract** — declared with this file's variables and **implemented once per backend entry
point**, not here. Returns whether the machine can run that backend at all; the
module-selection code calls it before committing to a renderer and falls through to the next
candidate when it says no.

**Notes** — It lives in this header because the header is what every backend already
includes, not because it has anything to do with console variables. A rebuild should
declare it with the renderer-selection interface instead.
