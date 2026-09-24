# src/Layers/xrRender/dxEnvironmentRender.cpp

> The renderer's filling of the sky and weather port: the two-layer cross-faded sky box, the cloud dome, and the texture plumbing that lets the weather system blend between two times of day.

**Needs** — [`dxEnvironmentRender.h`](dxEnvironmentRender.h.md) · [`Include/xrRender/EnvironmentRender.h`](../../Include/xrRender/EnvironmentRender.h.md) · [`Blender.h`](Blender.h.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`xrEngine/xr_efflensflare.h`](../../xrEngine/xr_efflensflare.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dxEnvironmentRender.h`](dxEnvironmentRender.h.md)
**Tier floor** — T1: it fills vertex and index buffers in place with a declared byte layout and rebinds texture surfaces underneath live texture objects.

## Purpose

The weather system owns *what the sky looks like at this moment*: it holds a list of time-of-day descriptors, picks the two that bracket the current clock, and interpolates every scalar between them. It does not know what a texture is. This file is the other half — it holds the sky and cloud geometry, the materials, and the two texture slots that the weather system's two bracketing descriptors are bound into so the shader can cross-fade them with a single weight.

The cross-fade is the load-bearing idea. Everything else here follows from it.

## State

```text
RECORD EnvDescriptorTextures     # one per time-of-day descriptor
  sky        : Texture            # the sky cube for this hour
  sky_env    : Texture            # the environment-reflection cube for this hour
  clouds     : Texture            # the cloud layer for this hour

RECORD EnvironmentRenderer
  sky_material, sky_geometry         : the sky box's material and vertex format
  clouds_material, clouds_geometry   : the cloud dome's

  sky_texture_list    : list<(stage, Texture)>   # rebuilt every interpolation
  clouds_texture_list : list<(stage, Texture)>

  sky0, sky1              : Texture   # NAMED placeholders, see below
  envmap0, envmap1        : Texture
  tonemap                 : Texture   # the luminance result, for exposure-correct sky

  sky0_stage, sky1_stage, clouds0_stage, clouds1_stage : int
  tonemap_stage_sky, tonemap_stage_clouds              : optional<int>
```

**Invariants**

- `sky0`, `sky1`, `envmap0`, `envmap1` and `tonemap` are **named placeholder textures**, created against the engine's reserved names (`$user$sky0`, `$user$sky1`, `$user$env_s0`, `$user$env_s1`, `$user$tonemap`). They own no image data of their own. Each frame, the two bracketing descriptors' real sky surfaces are *installed underneath* them. Every shader in the game that wants "the current sky" names the placeholder and gets whatever is installed. This indirection is what lets a hundred materials reference the sky without any of them knowing the weather.
- The surfaces installed under the sky placeholders must be **cube** surfaces. The sky is a cube map, sampled by direction, not by a screen coordinate.
- When the menu's post-processing is showing, the two environment-reflection placeholders are installed with *nothing*. That is deliberate: the menu blurs the world behind it and a live reflection cube would fight the blur.
- The texture stage indices are resolved **once, at device creation**, by asking each material's compiled first pass which stage its named sampler landed on. They are not constants. A material whose shader does not declare the sampler leaves the stage at zero and the entry is harmless; a material with no luminance sampler leaves the tone-map stage unset and the tone-map texture is simply not bound.

## `CBlender_skybox` — the sky material template

**Contract** — a template declared inside this file rather than in `blenders/`, because nothing else uses it. One pass, programs named `sky2`, depth-tested but **not** depth-writing, fogged off. Binds two sky samplers to the null texture — they are overwritten at draw time from the texture list — and the luminance result so the sky is exposed the same way the world is.

**Notes** — Binding `$null` and then overriding is not a workaround: it is how the pass reserves the sampler slot so that the stage lookup at device creation finds it. On a device with a fixed-function pipeline the engine instead names a material from the shipped data (`sky/skydome` with the `skybox_2t` texture set); the source notes that path blends the two layers by switching rather than fading, which is a known visual regression and not the intended behaviour.

## `render_sky(environment)`

**Contract** — draws the sky box, then the sun's lens flare. Fills both a vertex and an index buffer from the scratch pools, so it must run inside the frame. Leaves the far-plane projection mode restored.

```text
FUNCTION render_sky(env)
  push_far_projection()                 # the sky is drawn with the far clip pushed out

  transform = rotate_about_up(env.sky_rotation)
  transform.translation = camera_position        # the box follows the camera exactly

  colour = pack_rgba(env.sky_colour * 255, env.blend_weight * 255)
  # the ALPHA channel carries the cross-fade weight between the two sky layers

  copy the 20-triangle half-box index list into the index pool
  FOR v IN 0..11
    vertex[v] = { position = half_box_vertex[2v],
                  colour   = colour,
                  uv0 = uv1 = half_box_vertex[2v + 1] }   # the SECOND entry is a direction
  draw(triangles, 12 vertices, 20 triangles, sky_geometry, sky_material, sky_texture_list)

  pop_far_projection()
  draw the sun's lens flare (source and gradient, no flares)
```

**Invariants**

- The sky mesh is a **half box**: twelve vertices and twenty triangles, not the eight and twelve of a full cube. The lower half is collapsed to a skirt just below the horizon — the array pairs each position with a slightly-offset companion, and the pairs at the bottom sit at −1.0 and −1.01. The player never sees below the horizon (the world is there), so the bottom face is replaced by a thin band that closes the volume without wasting fill.
- Each mesh vertex occupies **two** entries in the source table: the position, then the texture coordinate, which is a *direction* (a three-component cube-map coordinate), not a two-component surface coordinate. Both sky layers are given the same direction; the shader fades between them.
- The box is translated to the camera's position every frame and never scaled. Its size is irrelevant because depth writing is off and the far plane is pushed out; what matters is that the camera is strictly inside it.
- The cross-fade weight rides in the vertex colour's **alpha**. There is no separate constant. A rebuild that moves it to a uniform must move it in the shader too.
- The lens flare is drawn *between* the sky and the clouds, with a depth-state dance around it: depth is disabled, re-enabled, then disabled again. That sequence exists because the flare's own drawing may set depth state through a path the draw stream does not observe, so the cached state is forced to a known value on both sides. A rebuild whose state tracking has no such hole writes one disable.

## `render_clouds(environment)`

**Contract** — draws the cloud dome. Returns immediately if no cloud material was created. Fills both pools from the environment's own cloud mesh, which the weather system owns.

```text
FUNCTION render_clouds(env)
  push_far_projection()
  transform = rotate_about_up(env.clouds_rotation) * scale(10, 0.4, 10)
  transform.translation = camera_position

  # two wind directions, 45° and 67.5° from north, packed into one colour
  wind0 = direction_from_heading(45°) ; wind1 = direction_from_heading(67.5°)
  packed_wind = pack_rgba(((wind0.x, wind0.z, wind1.z, wind1.x) * 0.5 + 0.5) * 255)
  cloud_colour = pack_rgba(env.clouds_colour * 255)      # alpha is the layer weight

  copy env.cloud_indices and env.cloud_vertices into the pools, giving every vertex
    the same two packed colours
  draw(triangles, ..., clouds_geometry, clouds_material, clouds_texture_list)
  pop_far_projection()
```

**Invariants**

- The dome is scaled **10 wide and 0.4 tall**. Clouds are a flattened dome, not a hemisphere; the ratio is what makes them read as a layer at altitude rather than a bowl.
- Two wind directions are packed into one four-channel colour, each as a pair of horizontal components mapped from [−1, 1] into [0, 1]. The shader scrolls two cloud layers along them at different rates, which is what keeps the sky from looking like one sliding texture. The component order is `(wind0.x, wind0.z, wind1.z, wind1.x)` — note the second pair is **swapped** relative to the first. Reproduce the order; the shader reads it positionally.
- The angles are fixed at 45° and 67.5°. They are not derived from anything in the weather data; they are authored constants.

## `on_device_create()` / `on_device_destroy()`

**Contract** — creates the two materials and their vertex formats, then resolves the four-or-six texture stage indices from the compiled first passes. Destroys everything and clears the installed surfaces on teardown. Does nothing at all when the process is running as a dedicated server.

**Invariants**

- The vertex formats are declared explicitly: the sky vertex is a position, a packed colour and **two three-component** texture coordinates; the cloud vertex is a position and **two** packed colours. Both are fixed byte layouts handed to the device.
- The installed surfaces must be uninstalled before the placeholders are destroyed, or the placeholder outlives a surface it still references.
- The stage resolution must happen after the materials compile and before the first draw. Because it reads the *compiled* pass, a material that failed to compile yields no stages and the sky draws untextured rather than crashing.

## `lerp(current, descriptor_a, descriptor_b)`

**Contract** — called once per frame by the weather system with the two bracketing descriptors. Rebuilds both texture lists from them, installs the two sky and two environment surfaces under the placeholders, and — on the oldest renderer generation only — pushes the fog colour and range into the device's fixed-function fog state.

**Notes** — Fog on the deferred path is computed in the shaders from the same weather values, so there is nothing to push; the source leaves the branch empty and says so. The name `lerp` is the weather system's word for "advance to this blend of these two descriptors", and the interpolation of the scalars has already happened by the time this is called — this half only swaps textures.

## `particles_systems_library()`

**Contract** — hands back the renderer's particle-effect library so the weather system can instantiate thunder and rain effects. A pure accessor; it is here because the port declares it and the renderer owns the library.
