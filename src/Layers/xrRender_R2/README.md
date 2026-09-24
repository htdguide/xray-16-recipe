# `src/Layers/xrRender_R2` — the deferred-shading path

This is the frame graph. Chapter 18 (`../xrRender`) supplies the pieces — the scene graph
and its visibility walk, the material/pass description loaded from data, the command
backend, the model pool, the light database. This chapter decides **what is drawn into
what, in which order, and why**, for both modern backends. The Direct3D 11 backend
(chapter 20) and the OpenGL backend (chapter 21) compile this same source twice; the
difference between them is confined to clip-space conventions, a handful of formats, and
which of the optional paths are wired. Anything that reads "one backend does X, the other
Y" in these pages is a *convention* difference, not a design difference.

Build order: this rests on chapters 1–18 and is consumed by 20 and 21. It never refers
forward. It reaches the graphics device only through the
[Graphics device seam](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device); the GPU
programs it drives ship as game data (see
[§5 Shaders](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)) and are never
written here. Each page therefore says what a pass *demands* of its program — which
targets are bound, which constants are set, which named element of the material it
selects — and stops there.

---

## The ideas you need before the twins make sense

**Deferred shading.** Geometry is rasterized once into a set of screen-sized buffers (the
*G-buffer*) that record, per pixel, everything lighting needs: where the surface is, which
way it faces, what colour it is and how shiny. Lighting then runs as a second, purely
2D pass per light, reading those buffers. The payoff is that geometry cost and light
count stop multiplying; the price is that transparency cannot participate, so everything
translucent is drawn later, forward, in a separate sorted pass.

**The G-buffer.** Three targets, all at back-buffer resolution, bound together:

| Target | Precision | Contents |
|---|---|---|
| position | four 16-bit floats | eye-space position `x,y,z`; the fourth channel carries material id |
| normal | four 16-bit floats | eye-space normal `x,y,z`; the fourth channel carries the hemispheric-ambient factor |
| albedo | four 16-bit floats, or four 8-bit when the device is weak | surface colour `r,g,b`; the fourth channel carries gloss |

The precisions are load-bearing. Eye-space *positions* rather than depth is a decision of
its era: it costs bandwidth but lets every lighting program work in one linear space with
no reconstruction, and 16-bit float has enough mantissa for eye-space at the ranges the
game uses. Ambient in the normal's fourth channel and gloss in the albedo's fourth
channel exist because three targets was the hardware limit — the "spare" channel of each
carries a scalar that would otherwise need a fourth target. A *g-buffer-optimized*
variant drops the normal target entirely and packs the normal into the position target's
spare channels, trading program work for bandwidth; the recipe keeps it because the pass
code branches on it everywhere.

**Material id.** A pixel does not carry a BRDF; it carries a small integer selecting one
of four material rows in a shared 3D lookup texture indexed by (N·L, N·H, material). The
lighting program is therefore the same for every surface — the material only changes
which slice it samples. This is why there is exactly one "accumulate a light" program per
light *shape*, not per surface type.

**The accumulator.** Lighting does not write to the frame; it adds into a separate
four-channel 16-bit-float target — `r,g,b` diffuse plus a specular scalar — cleared to
black once per frame. Every light, the sun, the emissive pass and the indirect bounces
all add into it. Tone mapping happens once at the end, on the sum. On hardware without
blending into 16-bit-float targets the same effect is reached with a second accumulator
and an explicit copy pass, which is why a "blend-copy" tail appears at the end of several
accumulation pages.

**The stencil is the light's bounding volume.** Each light is drawn as a closed convex
mesh — a sphere for a point light, a cone for a spot, a sphere cap for one face of a
cube-shadowed light. The mesh is rasterized twice with colour writes off, writing stencil
where the volume's back faces fail depth and clearing it again where the front faces do;
what survives is exactly the pixels inside the volume. The third draw runs the lighting
program only where the stencil equals that light's marker. The marker is not a flag but a
*counter* that advances by two per light, so consecutive lights do not need a stencil
clear between them — only when the counter would overflow the eight-bit (or seven-bit,
under multisampling, where the high bit marks edge pixels) range is a full-screen reset
issued. That trick is what makes two hundred lights per frame affordable.

**Shadow maps live in one atlas.** All shadow-casting spot lights share one square depth
texture. Each light asks for a square sub-rectangle whose side is chosen from its
screen-space importance; a first-fit packer places as many as will fit, the ones that fit
form one *batch*, and the whole batch renders into the atlas and is then accumulated
before the next batch reuses the same texture. The sun's cascades use a second texture,
an array with one slice per cascade.

**Phases run ahead of the frame.** Visibility for the main view, for each sun cascade and
for the rain shadow map are independent walks of the same scene graph, each with its own
command context. They are started during *calculate*, which happens before the frame's
drawing begins, and joined at the exact point in the frame graph where their output is
first needed. A phase is therefore a small state machine — start, calculate, render,
flush — and the frame graph is written as a sequence of joins.

**Two sun implementations ship.** Which one runs is decided at load time by *inspecting
the shipped shader source*: if the near-cascade sun program takes a screen-space quad it
is the older, two-pass trapezoidal sun; if it takes a light volume it is the newer
three-cascade sun. The engine must match whatever the installed game data expects, and
the data is the only place that answer exists.

---

## Pass order of one frame

Everything below happens inside `Render`, in this order. Reads and writes are named by
the targets above.

1. **Optional depth prefill** — a narrowed frustum, geometry only, colour writes off,
   filling depth so the G-buffer pass rejects early. Writes depth.
2. **Join the main visibility walk.** Clear depth and, when any of the advanced effects
   is on, the position target too.
3. **G-buffer fill** — the heads-up display first (drawn with its own near viewport
   range so it never intersects the world), then the static/dynamic scene graph, then the
   level-of-detail impostors, then the detail/grass layer. Writes position, normal,
   albedo, depth, and stencil `1` on every covered pixel.
4. **Occlusion queries for light volumes** — each visible light's bounding volume is
   drawn depth-tested with colour off against the just-filled depth, splitting the light
   set into *answered* and *still pending*. Reads depth.
5. **Wallmarks** (decals) — drawn into albedo with depth test, stencil-restricted to
   covered pixels.
6. **Multisample edge marking** — a full-screen pass that sets the high stencil bit on
   pixels whose samples disagree, so every later lighting pass can run once per pixel in
   the interior and once per sample only on edges.
7. **Rain wetness** — joins the rain shadow phase and rewrites normal, albedo and gloss
   in place for surfaces the sky can see. Reads the rain shadow map; writes normal and
   albedo.
8. **Sun** — joins the sun phase, which has already rendered its cascades and added each
   one into the accumulator; then blends the accumulated sun. Reads the cascade shadow
   array and the G-buffer; writes the accumulator.
9. **Emissive geometry** — self-lit surfaces drawn straight into the accumulator, marking
   stencil so later lights skip them.
10. **Lights that already answered their occlusion query**, then **lights still pending**.
    For each batch: fill the shadow atlas, then accumulate — unshadowed point, unshadowed
    spot, shadowed spots, then the volumetric (light-shaft) contribution of those spots
    into a separate target. Writes the accumulator and the volumetric target.
11. **Combine and post** — ambient occlusion, then the tone-mapped resolve of the
    accumulator against albedo into a low-dynamic-range target, the forward pass for
    everything deferred cannot express (translucent geometry, particles, the sky, faded
    portals), volumetric compositing, bloom with its embedded auto-exposure measurement,
    the distortion mask, depth of field and motion blur inside the final combine, and the
    optional film post-process. Presents.

Steps 11's composition lives in the backend chapters because it is the one part whose
target juggling differs; this chapter owns everything it calls.

---

## The render-target set and its lifetime

Created once when the device comes up, destroyed and rebuilt on every resolution change.
Nothing survives a frame boundary except the exposure pool.

| Target | Size | Lives |
|---|---|---|
| back buffers, back depth | screen | the device |
| position, normal, albedo | screen | filled in step 3, read until step 11 |
| multisample depth | screen | aliases the back depth when multisampling is off |
| accumulator (+ optional copy twin) | screen | cleared at first light of the frame, consumed at combine |
| generic 0/1/2 and generic | screen | scratch: post-process ping-pong, distortion mask, volumetric sum |
| shadow atlas (depth, and a colour surface only where the device demands one) | one square, configurable 1024…8192 | reused per light batch |
| sun cascade array | same square, one slice per cascade | one frame |
| rain shadow map | its own square | one frame |
| min/max shadow map | a quarter of the atlas side | one frame, only for light shafts |
| bloom 1 and 2 | 256×256 | ping-pong inside the bloom filter |
| luminance 64×64 and 8×8 | fixed | the auto-exposure reduction |
| luminance pool | 1×1, two per GPU | **survives frames** — this is the exposure feedback loop |
| ambient-occlusion scratch, half-depth | screen or half | one frame |

The luminance pool is the only cross-frame state in the whole set, and it is the reason
the screen brightens slowly when you walk out of a tunnel.

---

## Files

| File | Role |
|---|---|
| [`r2.h`](r2.h.md) | The renderer's public surface, the option record, and the phase-runner interface every off-frame walk implements |
| [`r2.cpp`](r2.cpp.md) | Device capability detection, option resolution, renderer lifecycle, per-frame statistics |
| [`r2_types.h`](r2_types.h.md) | Names of every render target, the material-pass element indices, and the shadow/bloom/exposure constants |
| [`r2_loader.cpp`](r2_loader.cpp.md) | Level load and unload: shaders, geometry buffers, visuals, sectors and portals, lights |
| [`r2_blenders.cpp`](r2_blenders.cpp.md) | Maps a material's class identifier to the deferred pass description that implements it |
| [`r2_R_calculate.cpp`](r2_R_calculate.cpp.md) | Per-frame preparation: screen-area thresholds, camera sector, light gathering, phase dispatch |
| [`r2_R_render.cpp`](r2_R_render.cpp.md) | **The frame graph**: the ordered pass list above |
| [`r2_R_lights.cpp`](r2_R_lights.cpp.md) | Shadow-atlas batching of lights, the accumulation order, and the indirect-bounce lights |
| [`SMAP_Allocator.h`](SMAP_Allocator.h.md) | The first-fit square packer that assigns atlas rectangles |
| [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) | Construction of the entire target set and every pass description; the G-buffer format decision |
| [`r2_rendertarget_accum_point_geom.cpp`](r2_rendertarget_accum_point_geom.cpp.md) | The sphere used as a point light's volume |
| [`r2_rendertarget_accum_spot_geom.cpp`](r2_rendertarget_accum_spot_geom.cpp.md) | The cone used as a spot light's volume, and the slice stack used for light shafts |
| [`r2_rendertarget_accum_omnipart_geom.cpp`](r2_rendertarget_accum_omnipart_geom.cpp.md) | The sphere cap used for one face of a cube-shadowed light |
| [`r2_rendertarget_draw_volume.cpp`](r2_rendertarget_draw_volume.cpp.md) | Dispatches a light to its volume mesh |
| [`r2_rendertarget_enable_scissor.cpp`](r2_rendertarget_enable_scissor.cpp.md) | Near-plane containment test that decides which face of a light volume is drawn, and the depth-bounds hint |
| [`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md) | Binding and first-touch clearing of the accumulator and the volumetric target |
| [`r2_rendertarget_phase_smap_S.cpp`](r2_rendertarget_phase_smap_S.cpp.md) | Atlas clear and per-light viewport for spot shadow rendering |
| [`r2_rendertarget_accum_reflected.cpp`](r2_rendertarget_accum_reflected.cpp.md) | Accumulating one indirect-bounce light |
| [`r2_rendertarget_phase_bloom.cpp`](r2_rendertarget_phase_bloom.cpp.md) | Bright-pass downsample and the separable Gaussian, plus where exposure measurement is spliced in |
| [`r2_rendertarget_phase_luminance.cpp`](r2_rendertarget_phase_luminance.cpp.md) | The three-step reduction to one exposure value and its adaptation rate |
| [`r2_rendertarget_phase_PP.cpp`](r2_rendertarget_phase_PP.cpp.md) | The film post-process: blur, desaturation, duality, animated grain, colour grading |
| [`r2_rendertarget_wallmarks.h`](r2_rendertarget_wallmarks.h.md) | A vestigial declaration |
| [`r3_rendertarget_phase_scene.cpp`](r3_rendertarget_phase_scene.cpp.md) | G-buffer bind, clear policy and the albedo work-around resolve |
| [`r3_rendertarget_phase_occq.cpp`](r3_rendertarget_phase_occq.cpp.md) | State for light-volume occlusion queries |
| [`r3_rendertarget_phase_smap_D.cpp`](r3_rendertarget_phase_smap_D.cpp.md) | Binding one sun cascade slice, or the rain map, for shadow rendering |
| [`r3_rendertarget_accum_point.cpp`](r3_rendertarget_accum_point.cpp.md) | Accumulating one point light: the stencil bound and the lighting draw |
| [`r3_rendertarget_accum_spot.cpp`](r3_rendertarget_accum_spot.cpp.md) | Accumulating one spot light, the shadow-lookup matrix, and the volumetric slice stack |
| [`r3_rendertarget_create_minmaxSM.cpp`](r3_rendertarget_create_minmaxSM.cpp.md) | Reducing a cascade to a min/max pyramid so light shafts can skip empty depth ranges |
| [`r3_rendertarget_mark_msaa_edges.cpp`](r3_rendertarget_mark_msaa_edges.cpp.md) | Marking multisample edge pixels in the high stencil bit |
| [`r3_rendertarget_phase_ssao.cpp`](r3_rendertarget_phase_ssao.cpp.md) | Ambient occlusion at half resolution and the depth downsample it needs |
| [`render_phase_sun.cpp`](render_phase_sun.cpp.md) | **The cascaded sun**: split sizes, chained cuboid fitting, texel snapping, per-cascade accumulation |
| [`render_phase_sun_old.cpp`](render_phase_sun_old.cpp.md) | The legacy two-region sun with trapezoidal projection and focus refit |
| [`r2_R_sun_support.h`](r2_R_sun_support.h.md) | The convex-volume math both suns stand on: caster hulls, clipping, frustum extrema |
| [`r3_R_rain.cpp`](r3_R_rain.cpp.md) | The downward "rain light" and its shadow map, used for wetness rather than shadowing |
| [`r3_rendertarget_draw_rain.cpp`](r3_rendertarget_draw_rain.cpp.md) | Rewriting normal, albedo and gloss of sky-exposed surfaces to look wet |
