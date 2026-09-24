# src/Layers/xrRender/Blender_Recorder_StandartBinding.cpp

> The published vocabulary of shader constant names: every name a shipped shader may spell to get a value filled in for it each frame, and the producer behind each one.

**Needs** — [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_xform.h`](R_Backend_xform.h.md) · [`R_Backend_hemi.h`](R_Backend_hemi.h.md) · [`R_Backend_tree.h`](R_Backend_tree.h.md) · [`xrEngine/IGame_Persistent.h`](../../xrEngine/IGame_Persistent.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: each producer writes directly into a constant buffer through the backend, and the projection-adjustment matrices differ by device convention.

## Purpose

This is the **name contract between the engine and the shipped shader data**, and it is the reason a material library authored in 2007 still renders. A shader source file that declares a constant called `m_WVP` gets the world-view-projection matrix; one that declares `fog_color` gets the current weather's fog colour. Nothing in the material library says so — the binding is installed into every pass automatically, and a pass whose programs happen not to declare a name simply does not receive it.

A rebuild must reproduce **every name in this file, spelled exactly**, or the shipped shaders will read uninitialized constants. The *values* behind the names are the rebuild's own business; the names are frozen.

## State

```text
RECORD ConstantBinder               # one per name; a singleton, shared by every pass
  setup(command_list, constant)     # called once per pass activation, per frame

RECORD CachedBinder : ConstantBinder    # the common shape for per-frame world values
  last_frame : int
  value      : vector4
  # invariant: value is recomputed at most once per frame, then replayed
  # into every pass that asked for it
```

**Invariants** — a binder is a stateless singleton *except* for the cached ones, which memoize one frame's value. The cache is keyed on the frame counter alone, which means it is correct only while one frame is being built by one thread at a time; a rebuild with parallel command-list recording must make the cache per-context or move it to a frame-start computation.

There is a real defect worth naming: the cached binders never initialize their frame marker, so on the very first frame the memo may be considered valid and a stale value is published. In practice the first frame is not shown. A rebuild should initialize it to "never".

## The vocabulary

### Transforms

```text
m_W      world
m_invW   inverse world
m_V      view
m_P      projection
m_WV     world * view
m_VP     view * projection
m_WVP    world * view * projection
```

Each is served straight from the backend's transform cache, which recomputes the products only when one of the factors changes. The name list is the *complete* set of products the shipped shaders use; there is no general "multiply these two" facility.

### Texture-coordinate generation

```text
m_texgen    world*view*projection, adjusted to texture space
mVPTexgen   view*projection,       adjusted to texture space
```

**Notes** — The adjustment matrix maps clip space (−1..1 on both axes) to texture space (0..1) and **flips the vertical axis on one device family and not the other**, because the two graphics APIs disagree about whether texture space starts at the top or the bottom. This is exactly one sign in one matrix element, and getting it wrong inverts every projected shadow and every screen-space lookup in the frame. It is the single most portable-looking and least portable line in the renderer.

### Tree and detail wind animation

```text
m_xform_v  m_xform  consts  wave  wind  c_scale  c_bias  c_sun
```

Served from the backend's tree-animation block. These are the constants the wind model publishes to the tree and grass shaders: the instance transform in two spaces, the per-instance colour scale and bias, the sun contribution, and the wave/wind parameters that drive the vertex displacement. Their meaning is in [`R_Backend_tree.cpp`](R_Backend_tree.cpp.md).

### Hemisphere lighting and the material lookup

```text
L_material            selects a row of the shipped material-response lookup
hemi_cube_pos_faces   per-object ambient occlusion, the three positive axes
hemi_cube_neg_faces   ...and the three negative
```

**Notes** — The hemisphere cube is this engine's per-object ambient term: six scalars saying how much sky each face direction sees, sampled from the level's baked lighting at the object's position. A dynamic model carries six numbers instead of a lightmap, which is why characters sit correctly in a dark interior. Reproducing it matters more than it looks — without it, dynamic objects render at full ambient everywhere and look pasted on.

### Fog

```text
fog_plane    a view-space plane whose signed distance is the fog factor
fog_params   (-near/range, 1/range, 1/range, 1/range)
fog_color    the current weather's fog colour
```

```text
FUNCTION fog_plane_value()
  # the far plane of the current frustum, extracted from the combined
  # transform and normalized: (row4 + row3), negated
  plane := -(column 4 of full_transform + column 3 of full_transform)
  plane := plane * (-1 / length(plane.xyz))
  near  := environment.fog_near
  range := environment.fog_far - near
  # pre-divide so the shader's fog factor is one dot product
  RETURN (-plane.xyz / range,  1 - (plane.w - near) / range)
```

**Notes** — Fog is computed from a *plane* rather than from distance to the camera so that it costs one dot product in the vertex program and produces planar (not radial) fog, which is what the art is authored against. The pre-division folds the near/far range into the plane so the shader does no division at all. A rebuild that fogs by radial distance will render a visibly different horizon.

### Time, camera, weather

```text
timers          (t, t*10, t/10, sin t)  where t is global seconds since start
eye_position    camera position, w = 1
eye_direction   camera forward,  w = 0
eye_normal      camera up,       w = 0
L_sun_color     current weather's sun colour
L_sun_dir_w     sun direction, world space
L_sun_dir_e     sun direction, view space, renormalized
L_hemi_color    sky/hemisphere colour, with its own intensity in w
L_ambient       ambient colour, with the weather blend weight in w
screen_res      (width, height, 1/width, 1/height)
```

**Notes** — `timers` packing four rates of the same clock into one constant is the kind of decision that looks arbitrary and is not: a shipped shader animating at some rate picks the component closest to it rather than multiplying, which on the oldest pixel-shader model saved an instruction it did not have. The ×10 and ÷10 factors are therefore part of the data contract — an effect authored against `timers.y` scrolls ten times faster than one authored against `timers.x`.

`L_ambient.w` carrying the *weather blend weight* rather than an alpha is the sort of overload that is invisible until it breaks. The environment system cross-fades between two weather sets; that weight rides along in a component nothing else needed.

### Per-object and script-driven

```text
m_hud_params       first-person-view tuning, set by the game layer
m_script_params    a free vector the scripts may write, for modded effects
m_blender_mode     a mode selector the game layer switches per draw
m_obj_camo_data    per-object camouflage
m_obj_custom_data  per-object free vector
m_obj_entity_data  per-object entity state
```

**Notes** — The three object slots are not filled here. Their binders merely *record where the constant lives* in the backend, so that the draw loop can write a value per object without re-resolving a name. That is the pattern to copy: name resolution happens once at material compile time and the per-draw path writes to a slot.

These six are additions by the modding community rather than the original engine, and they exist because the shipped shader vocabulary is otherwise closed. A rebuild aiming only at the retail games can omit them; one that wants the mod ecosystem cannot.

### Detail-texture tiling

`dt_params` is bound to the *detail scaler* the compile step resolved out of the texture-description database — the per-base-texture tiling factors of its detail partner. It is bound whenever a scaler was found, even if detail texturing was subsequently disabled by a quality setting, because a shader may apply detail implicitly.

### Everything else

After the fixed list, the resource manager's table of externally registered (name, producer) pairs is installed as well. That is the extension point: a renderer backend publishes its own names — cascade matrices, screen-space ambient occlusion parameters, whatever its shaders need — by registering them there, and the shared material compiler binds them without knowing what they are.

## `SetMapping`

**Contract** — installs every binding above into the pass currently being recorded. Called by both recorders at the end of each pass, before the constant table is interned. Binding a name the pass does not declare is a no-op, so the cost is one table lookup per name per pass at load time and nothing at draw time.
