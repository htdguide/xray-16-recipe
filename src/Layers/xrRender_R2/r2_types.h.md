# src/Layers/xrRender_R2/r2_types.h

> The shared vocabulary of the deferred path: the name of every render target, the index
> of every material-pass variant, and the constants that fix shadow, bloom and exposure
> sizes.

**Needs** — [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`blender_bloom_build.cpp`](../xrRender/blenders/blender_bloom_build.cpp.md) · [`blender_combine.cpp`](../xrRender/blenders/blender_combine.cpp.md) · [`blender_deffer_aref.cpp`](../xrRender/blenders/blender_deffer_aref.cpp.md) · [`blender_deffer_model.cpp`](../xrRender/blenders/blender_deffer_model.cpp.md) · [`blender_light_direct.cpp`](../xrRender/blenders/blender_light_direct.cpp.md) · [`blender_light_direct_cascade.cpp`](../xrRender/blenders/blender_light_direct_cascade.cpp.md) · [`blender_light_mask.cpp`](../xrRender/blenders/blender_light_mask.cpp.md) · [`blender_light_point.cpp`](../xrRender/blenders/blender_light_point.cpp.md) · [`blender_light_reflected.cpp`](../xrRender/blenders/blender_light_reflected.cpp.md) · [`blender_light_spot.cpp`](../xrRender/blenders/blender_light_spot.cpp.md) · [`blender_luminance.cpp`](../xrRender/blenders/blender_luminance.cpp.md) · [`blender_ssao.cpp`](../xrRender/blenders/blender_ssao.cpp.md) · [`dx11HDAOCSBlender.cpp`](../xrRender/blenders/dx11HDAOCSBlender.cpp.md) · [`dx11MSAABlender.cpp`](../xrRender/blenders/dx11MSAABlender.cpp.md) · _and 26 more_
**Tier floor** — T3: a table of names and numbers; nothing here touches the device.

## Purpose

Every render target in this chapter is addressed by *name*, not by handle, because a
material pass description loaded from game data names its texture inputs as strings and
the resource system resolves those against the same table. The names must therefore be
frozen: `$user$position`, `$user$accum`, `$user$smap_depth` and the rest appear verbatim
in shipped shader configuration. This file is where that namespace is declared once, so
that the creating side (target construction) and the consuming side (pass setup) cannot
drift apart.

It also holds the numbers that are decisions rather than tuning: which pass of a material
is "shadow" versus "detailed" versus "coarse", how large a shadow rectangle may get, and
how big the bloom and exposure reduction chains are.

## State

Stateless — a declaration file. The records below are the vocabulary it declares.

```text
# Render target names. Frozen: shipped shader configuration names them as strings.
RECORD TargetNames
  base_<n>, base_depth       # the swap chain's colour buffers and its depth buffer
  depth, msaa_depth          # scene depth; the second is multisampled, else an alias
  position, normal, albedo   # the G-buffer
  accum, accum_temp          # lighting accumulator; the twin exists only without fp blend
  generic0, generic1         # low-dynamic-range scratch, one per post-process stage
  generic0_r, generic1_r     # multisampled twins; aliases when multisampling is off
  generic2                   # volumetric (light shaft) accumulation
  generic                    # a third scratch, used as the pre-present image
  ssao_temp, half_depth      # ambient-occlusion working set
  bloom1, bloom2             # bloom ping-pong
  lum_t64, lum_t8            # exposure reduction stages
  tonemap_src, tonemap       # previous frame's and this frame's exposure, as textures
  luminance_<n>              # the 1x1 exposure pool, persists across frames
  smap_surf, smap_depth      # the spot shadow atlas (colour surface only if demanded)
  smap_rain                  # the downward rain shadow map
  smap_depth_minmax          # quarter-size min/max pyramid of a cascade
  material                   # the 3D BRDF lookup, indexed (N.L, N.H, material id)
  jitter_<n>, jitter_mipped  # dither/rotation noise
  async_ss                   # host-visible copy for screenshots
```

```text
# Which element of a material's pass list a draw selects.
ENUM SurfaceElement            # for world and model surfaces
  NORMAL_HQ = 0   # full detail: parallax and detail textures, used inside a distance
  NORMAL_LQ = 1   # the same surface without the expensive extras
  SHADOW    = 2   # depth only, used when filling any shadow map

ENUM LightElement              # for a light's accumulation material
  FILL       = 0  # writes the light's colour mask into the shadow atlas (translucency)
  UNSHADOWED = 1
  NORMAL     = 2  # shadowed, atlas rectangle smaller than the whole atlas
  FULLSIZE   = 3  # shadowed, rectangle is the whole atlas: the scale/offset is identity
  TRANSLUENT = 4  # shadowed through a coloured mask

ENUM MaskElement               # for the stencil-marking material
  SPOT, POINT, DIRECT, ACCUM_VOL, ACCUM_2D, ALBEDO

ENUM SunSubPhase
  NEAR = 0, MIDDLE = 1, FAR = 2   # cascades, near to far
  LUMINANCE = 3                   # the optional filtered-sun resolve
  NEAR_MINMAX = 4                 # near cascade sampled through the min/max pyramid
  RAIN_SMAP = 5                   # the rain map reuses the direct-shadow machinery
```

## Constants

**Contract** — these are the numbers a rebuild must reproduce to get the original's look
and performance; each is explained rather than restated.

```text
shadow_near_plane      = 0.1    # near plane of every spot shadow projection
shadow_size_min        = 32     # smallest atlas rectangle a light may be given
shadow_size_optimal    = 768    # the size a light of "reference" screen size asks for
shadow_size_max        = 1536   # cap, so one light cannot monopolise the atlas
material_lookup        = 128 x 256 x 4   # (N.L) x (N.H) x material id
jitter_tile            = 64     # side of one noise tile
jitter_tile_count      = 5      # distinct rotations; ambient occlusion cycles through them
bloom_target           = 256 x 256
exposure_reduction     = 16     # the 8x8 stage's grouping; see the luminance twin
```

**Notes** — the shadow sizes are a *range*, not a size: a light's rectangle is derived
from its projected screen area, clamped into `[min, max]`, and the optimal value is the
midpoint the derivation is scaled against. The lookup texture is 128 wide because the
diffuse term along `N·L` is nearly linear and quantizes well, and 256 tall because the
specular term along `N·H` is not — halving the wrong axis shows as banding on shiny
surfaces. Four materials is the count the shipped art uses; the fourth channel of the
G-buffer's position target only has room for a small integer anyway.

## `gloss_from_colour`

**Contract** — maps a light's colour to the specular scale carried alongside it. Pure,
no allocation.

```text
FUNCTION gloss_from_colour(r, g, b) -> real
  v = (r + g + b) / 3
  IF v < 1 THEN v = v ^ (2/3)      # compress: dim lights keep more relative specular
  RETURN gloss_factor * v          # gloss_factor is a user setting
```

**Notes** — the cube-root-squared curve below unity is the whole point: specular
highlights from weak lights would otherwise vanish before their diffuse contribution did,
which reads as flat plastic in the game's many dim interiors. Above unity the curve is
left linear so that an over-bright light does not gain specular faster than diffuse.
