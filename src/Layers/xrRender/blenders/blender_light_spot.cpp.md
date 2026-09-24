# src/Layers/xrRender/blenders/blender_light_spot.cpp

> The template that adds one cone light's contribution to the accumulator, plus the pass that makes its beam visible in the air.

**Needs** — [`blender_light_spot.h`](blender_light_spot.h.md) · [`blender_light_point.cpp`](blender_light_point.cpp.md) · [`blender_light_direct.cpp`](blender_light_direct.cpp.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — [`blender_light_spot.h`](blender_light_spot.h.md)
**Tier floor** — T2: a decision tree that emits pass descriptions.

## Purpose

The cone-light counterpart of [`blender_light_point.cpp`](blender_light_point.cpp.md). It shares that file's element namespace (`L_FILL`, `L_UNSHADOWED`, `L_NORMAL`, `L_FULLSIZE`, `L_TRANSLUENT`) and its binding set; only the decisions that differ are recorded here.

Spot lights carry most of the game's atmosphere — every lamp, flashlight and anomaly glow is one — and unlike point lights they are the lights that get a *volumetric* pass, because a cone in fog is the shape a player reads as a light beam.

## `CBlender_accum_spot.compile(context)`

**Contract** — emits one accumulation pass for the requested element, using the `accum_spot_*` program family.

**Differences from the point-light template, each load-bearing**

- The light's projected texture is sampled **clamped**, not wrapped. A spot's texture is its cone profile: wrapping it would tile the profile outside the cone and paint light where the cone is not.
- `L_NORMAL` and `L_FULLSIZE` name *different* programs (`accum_spot_normal` and `accum_spot_fullsize`). For a spot light the two differ in how the shadow lookup is scaled, because a reduced-size shadow map for a cone occupies a sub-rectangle of the shared shadow target and the coordinates must be scaled into it; a full-size one uses the whole target.
- `L_TRANSLUENT` reuses the *fullsize* program, and rebinds the light's texture slot to the shadow map's colour surface, exactly as the point template does.

```text
FUNCTION compile(C)
  base_compile(C)
  blend = device_can_blend_accumulator_format
  dest  = blend ? ONE : ZERO

  SWITCH C.element
    CASE L_FILL:       copy C.textures[0] into the mask target, no depth
    CASE L_UNSHADOWED: pass("accum_volume","accum_spot_unshadowed"),
                       s_lmap <- C.textures[0] CLAMPED
    CASE L_NORMAL:     pass("accum_volume","accum_spot_normal"),   + s_smap + jitter
    CASE L_FULLSIZE:   pass("accum_volume","accum_spot_fullsize"), + s_smap + jitter
    CASE L_TRANSLUENT: pass("accum_volume","accum_spot_fullsize"),
                       s_lmap <- shadow COLOUR surface, + s_smap + jitter
  # every case: depth test off, additive-or-replace by `blend`, volume geometry
```

## `CBlender_accum_spot_msaa.compile(context)`

**Contract** — the same, with `_msaa`-suffixed programs and a baked sample index; the mechanism is described once in [`blender_light_direct.cpp`](blender_light_direct.cpp.md).

## `CBlender_accum_volumetric_msaa.compile(context)`

**Contract** — one pass, element zero only, that integrates the light's contribution through the air inside its cone rather than on surfaces.

```text
FUNCTION compile(C)
  C.pass(vertex="accum_volumetric", pixel="accum_volumetric_msaa",
         fog=false, depth_test=false, depth_write=false)
  C.texture("s_lmap",  C.textures[0])          # the cone profile
  C.texture("s_smap",  shadow depth target)    # so the beam is interrupted by occluders
  C.texture("s_noise", "fx/fx_noise")          # animates the scattering
  C.end()
```

**Invariants** — the noise texture is referenced by a fixed path in the shipped data, `fx/fx_noise`. It is not a parameter and not a render target; it must exist in the mounted filesystem or the pass fails to build.

**Notes** — The beam is shadowed by the same shadow map the surface pass uses. That is what makes a flashlight beam cut off behind a doorframe rather than shining through it, and it is why the volumetric pass must run after the shadow map for that light is rendered and before it is reused for the next light.
