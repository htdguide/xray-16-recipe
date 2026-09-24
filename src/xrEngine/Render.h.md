# src/xrEngine/Render.h

> The frame-graph boundary: everything the engine may ask of a graphics backend, and the three resource kinds it holds handles to.

**Needs** — [`Render.cpp`](Render.cpp.md) · [`vis_common.h`](vis_common.h.md) · [`xrCDB/Frustum.h`](../xrCDB/Frustum.h.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md) · [`IRenderable.h`](IRenderable.h.md) · [`IPerformanceAlert.hpp`](IPerformanceAlert.hpp.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`xrAPI.h`](../Include/xrAPI/xrAPI.h.md) · [`UIShader.h`](../Include/xrRender/UIShader.h.md) · [`D3DXRenderBase.h`](../Layers/xrRender/D3DXRenderBase.h.md) · [`WallmarksEngine.cpp`](../Layers/xrRender/WallmarksEngine.cpp.md) · [`WallmarksEngine.h`](../Layers/xrRender/WallmarksEngine.h.md) · [`CameraManager.cpp`](CameraManager.cpp.md) · [`Device_Initialize.cpp`](Device_Initialize.cpp.md) · [`Device_create.cpp`](Device_create.cpp.md) · [`Device_destroy.cpp`](Device_destroy.cpp.md) · [`Device_overdraw.cpp`](Device_overdraw.cpp.md) · [`Environment.cpp`](Environment.cpp.md) · [`Environment_misc.cpp`](Environment_misc.cpp.md) · [`Environment_render.cpp`](Environment_render.cpp.md) · [`FDemoPlay.cpp`](FDemoPlay.cpp.md) · _and 24 more_
**Tier floor** — T1: the interface hands out handles whose release order is defined and passes scene data by reference into device-facing code every frame

## Purpose

This is the pluggable seam of the whole engine. The repository ships two fillings — an
OpenGL renderer used everywhere and a Direct3D 11 renderer used on Windows — and selects
one by name at startup, falling back to the next candidate when device creation fails.

The interface is *wide* because it is drawn at the wrong-looking place: it is not a
draw-call boundary but a **frame-graph** boundary. The engine hands the backend a scene and
a camera and receives a presented frame; everything between — visibility, sorting,
shadowing, lighting, post-processing — is the backend's. A narrow draw-call interface would
have forced the engine to own the frame structure, and the two backends structure their
frames differently.

This is an abstract interface header with no implementation file of substance, so the
contract here *is* the specification a filling must satisfy.

## State

The interface itself carries four tunables and one datum, all public and written from
outside:

```text
RECORD IRender (the mutable part)
  hq_skinning : bool      # use the higher-quality skinning path
  skinning    : int       # skinning mode, set as a shader compile option
  msaa_sample : int       # multisample count
  smap_size   : int       # shadow-map edge length in texels
  view_base   : Frustum   # the main camera's frustum, published for anyone who culls
```

`view_base` being public is how non-render code (the sound system's occlusion, the
scheduler's importance estimate, debug tooling) culls without asking the renderer.

## `IRender_Light`

**Contract** — a handle to one dynamic light owned by the backend. Configured entirely
through setters, because a light's representation differs per backend and the engine must
never see it. Lights are reference-counted resources; the last release destroys them
through the backend (see [`Render.cpp`](Render.cpp.md)).

```text
ENUM LightType  DIRECT, POINT, SPOT, OMNIPART, REFLECTED

INTERFACE Light
  set_type(LightType)
  set_active(bool) / get_active() -> bool
  set_shadow(bool)
  set_volumetric(bool), set_volumetric_quality(real),
  set_volumetric_intensity(real), set_volumetric_distance(real)
  set_indirect(bool)                 # optional: a filling may ignore it
  set_position(vector3)
  set_rotation(direction, right : vector3)
  set_cone(angle : real)             # spot only
  set_range(real)
  set_virtual_size(real)             # apparent source size, for soft shadows and glare
  set_texture(name : text)           # projected cookie
  set_color(...)
  set_hud_mode(bool) / get_hud_mode() -> bool
```

`OMNIPART` is a point light restricted to one face of its cube — the engine splits an omni
light into parts so each can be culled and shadowed separately. `REFLECTED` is a light the
backend generated itself as a bounce; the engine creates it but does not own its placement.
`hud_mode` marks a light that illuminates only the first-person overlay, which is rendered
in its own space with its own near plane.

## `IRender_Glow`

**Contract** — a handle to one billboard glow: a position, a direction, a radius, a texture
and a colour. Separate from lights because a glow is *visibility-tested* rather than lit —
the backend occlusion-queries each one and fades it by how much of it is visible.

## `IRender_ObjectSpecific`

**Contract** — the per-object lighting cache created lazily by every renderable (see
[`IRenderable.cpp`](IRenderable.cpp.md)). It accumulates the light reaching an object's
position from three sources, traced over several frames rather than each frame, and answers
queries about it.

```text
INTERFACE ObjectSpecific
  force_mode(mask)                     # re-trace now: any of LIGHTS, SUN, HEMI
  get_luminocity()           -> real   # total
  get_luminocity_hemi()      -> real   # sky contribution, scalar
  get_luminocity_hemi_cube() -> real[6]  # sky contribution per axis direction
```

**Notes** — the cube form exists because consumers care about direction, not just amount:
the rain effect reads it to decide whether the listener is under a roof, and the AI uses it
to decide whether a creature is standing in light. The six entries are the axis-aligned
faces; the downward one is routinely skipped by consumers because it sees the ground.

The engine's own spelling is *luminocity*; it is in the exported script surface.

## `IRender` — capability and identity

**Contract** — the backend declares what generation it is and which API it sits on, so the
game can enable or refuse features without naming a backend.

```text
ENUM Generation  R1 = 1, R2 = 2        # R1: forward, fixed-function era. R2+: deferred
ENUM BackendAPI  D3D9, D3D10, D3D11, OpenGL

get_generation() -> Generation
generation_is_r1() / is_r2() / is_r2_or_higher() -> bool
get_backend_api() -> BackendAPI
is_sun_static()   -> bool              # can the sun move, or is lighting baked?
get_dx_level()    -> int               # finer feature level within a generation
```

**Notes** — generation is compared with `>=`, so a new deferred backend claims R2 and
inherits every game-side decision made for the deferred path. That ordering is the whole
point of the enumeration being numeric.

## `IRender` — lifecycle

**Contract** — creation and destruction are split into a device half and a resource half,
because a device may be lost and rebuilt while the engine's object graph stays alive.

```text
create() / destroy()
reset_begin() / reset_end()             # bracket a device reset: release, then rebuild
on_device_create(shader_name) / on_device_destroy(keep_textures : bool)
create(window, out width, out height, out half_width, out half_height)
reset(window, out width, out height, out half_width, out half_height)
obtain_required_window_flags(inout flags)   # the backend states what the window must be
setup_states()
get_device_state() -> {NORMAL, LOST, NEED_RESET}
```

`obtain_required_window_flags` is the inversion that lets the windowing seam stay generic:
the window is created once with flags the *backend* asked for, rather than the engine
guessing what a graphics context needs.

`on_device_destroy` takes a flag to keep textures, because a resolution change needs the
device rebuilt but not the texture set re-read from disk — re-reading is seconds of work.

**Invariants** — the half-width and half-height outputs are the render target's dimensions
halved, returned rather than derived because the engine uses them in every screen-space
transform and the backend may round differently.

## `IRender` — level lifecycle

```text
level_load(stream) / level_unload()
```

**Contract** — the backend reads its own chunks out of the level file: the sector and portal
topology, the light maps, the static geometry with its material assignment. The engine does
not parse them. This is why the level file is passed as an open stream rather than as parsed
data.

## `IRender` — the scene

**Contract** — the engine adds things to the frame and then asks for it.

```text
add_visual(context_id, root : Renderable, visual, transform)   # no culling: caller decided
add_static_wallmark(material, position, size, triangle, vertices)
add_skeleton_wallmark(transform, skeleton, material, start, direction, size)
clear_static_wallmarks()
calculate()          # visibility, sorting, shadow setup
render()             # the world
render_menu()        # the menu, which has its own much simpler path
before_world_render() / after_world_render()   # game-module hooks around the world pass
```

`add_visual` explicitly performs **no** culling — the caller has already decided this thing
is visible. It exists for attachments and for objects the game knows are on screen, and
using it for a general object defeats the whole visibility system.

A *wallmark* is a decal. There are two kinds and they cannot share a path: a static
wallmark is clipped against one triangle of the level's collision mesh and baked into a
static buffer; a skeleton wallmark is projected onto an animated mesh and must be re-skinned
every frame with its host. `before_world_render` and `after_world_render` bracket the world
pass so the game module can inject its own full-screen work before the interface is drawn.

## `IRender` — occlusion queries

```text
occ_visible(vis_data)  -> bool
occ_visible(box)       -> bool
occ_visible(polygon)   -> bool
```

**Contract** — asks whether something is visible *this frame*, given whatever the backend
knows: hierarchical occlusion, the software occluder buffer, or nothing at all. A backend
that cannot answer must return true, because a false negative deletes geometry from the
frame. The three overloads exist because the callers hold three different descriptions and
converting them all to one would cost more than the test.

## `IRender` — models and resources

```text
model_create(name, optional stream) / model_create_child(name, stream)
model_create_particles(name)
model_duplicate(visual) / model_delete(visual, discard : bool)
model_logging(bool) / models_prefetch() / models_clear(complete : bool)

deferred_load(bool)
resources_deferred_upload() / resources_deferred_unload()
resources_get_memory_usage() -> (base, count, lightmaps, lightmap_count)
resources_destroy_necessary_textures() / resources_store_necessary_textures()
resources_dump_memory_usage()
```

**Contract** — models are handles, reference-counted by name inside the backend;
`model_duplicate` shares the immutable parts and copies only what an instance must own (its
animation state). `model_delete`'s discard flag forces the shared asset out rather than
letting the cache keep it, used when a level unloads.

*Deferred load* is the mechanism behind the precache: while it is on, requesting a resource
records the request and returns a placeholder, so a level's whole working set is discovered
without stalling; `resources_deferred_upload` then uploads everything at once. This is what
makes the precache frames in the frame loop (see [`device.cpp`](device.cpp.md)) worth
running.

`resources_store_necessary_textures` and its destroy counterpart bracket the precache: the
set of textures actually touched during precache is remembered, and everything else is
dropped when the precache ends.

## `IRender` — shaders

```text
shader_compile(name, source_stream, entry_point, target, flags, out result) -> status
get_shader_path() -> text
```

**Contract** — shader *source* ships with the game data (see the system requirements, §5),
so compilation happens at load time, not at build time. The backend owns a compiled-blob
cache keyed by the source hash and the macro set. `get_shader_path` names the subdirectory
this backend's dialect lives in, which is how one data tree serves several backends.

## `IRender` — gamma and presentation

```text
set_gamma(real) / set_brightness(real) / set_contrast(real) / update_gamma()
begin() / clear() / end() / clear_target()
set_cache_xform(view, projection)
on_camera_updated()
screenshot(mode, name)
set_post_process_params(info)
```

**Contract** — the three gamma-family setters stage values and `update_gamma` applies them
in one act, because applying them separately makes the screen flash three times.
`begin`/`end` bracket the frame; `end` is where presentation happens.

```text
ENUM ScreenshotMode
  NORMAL        # compressed photo, into the screenshot directory; name ignored
  FOR_CUBEMAP   # lossless, name used as a suffix; six of these make a cube
  FOR_GAMESAVE  # block-compressed, name is a full path; the save-game thumbnail
  FOR_LEVELMAP  # lossless, name used as a suffix; feeds the in-game map
```

The four modes differ in format *and* in how the name is interpreted, which is why one
enumeration covers them rather than four functions: every caller is a console command with
the same shape.

## `IRender` — render contexts

```text
ENUM RenderContext  NONE = -1, PRIMARY, HELPER
get_current_context() / make_context_current(context)
```

**Contract** — two contexts: the primary one that produces the frame, and a helper used for
work that must touch the device off the frame path (uploading a resource discovered by a
loader thread, building a cubemap). Switching is explicit and scoped; the scoped form is in
[`Render.cpp`](Render.cpp.md).

## `IRender` — statistics

**Contract** — the backend owns a block of per-frame timers and counters and dumps them
into the statistics overlay itself, complaining through the performance-alert port when a
number is bad. The block is part of the interface because the overlay prints it and the
frame pacing reads the totals.

```text
RECORD RenderStatistics
  # timers, each accumulated and reset per frame
  culling          # portal traversal, frustum culling, per-object submission
  animation        # skeleton evaluation
  skinning
  primitives       # actual submission
  wait             # blocked on the device: query results and similar
  wait_sync        # blocked on the frame limiter
  render_targets
  detail_visibility / detail_render / detail_cache    # the grass-and-debris layer
  wallmarks
  hud
  glows / lights / projectors
  shadows_calc / shadows_render
  # counters, reset per frame
  detail_count, static_wallmark_count, dynamic_wallmark_count, wallmark_triangles
  occlusion_queries, occlusion_culled
```

**Invariants** — timers are started at frame start and ended at frame end in matched pairs;
counters are zeroed at frame start only. The split matters: a timer that is ended without
being started reports a garbage interval, which is why the two lifecycle calls enumerate
every member rather than looping.

## `DeviceState`

```text
ENUM DeviceState  NORMAL, LOST, NEED_RESET
```

**Contract** — the three-state device-loss protocol the frame loop is written against.
`LOST` means do not draw and try again later; `NEED_RESET` means the device can be
rebuilt now. The frame loop's handling is in [`device.cpp`](device.cpp.md).

## `xrImTextureData`

**Contract** — a texture handed to the debug overlay toolkit: an opaque identifier the
toolkit passes back at draw time, plus the dimensions, since the toolkit needs them for
layout and cannot ask the device. Obtained by name through `get_imgui_texture_id`, which is
how debug tools display game textures without touching the resource system.
