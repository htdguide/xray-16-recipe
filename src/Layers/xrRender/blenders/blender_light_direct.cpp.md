# src/Layers/xrRender/blenders/blender_light_direct.cpp

> The templates that add the sun's contribution to the light accumulator, one shadow cascade at a time, plus the sample-resolved and volumetric variants of the same.

**Needs** — [`blender_light_direct.h`](blender_light_direct.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`blender_light_direct.h`](blender_light_direct.h.md) · [`blender_light_direct_cascade.cpp`](blender_light_direct_cascade.cpp.md) · [`blender_light_mask.cpp`](blender_light_mask.cpp.md) · [`blender_light_point.cpp`](blender_light_point.cpp.md) · [`blender_light_reflected.cpp`](blender_light_reflected.cpp.md) · [`blender_light_spot.cpp`](blender_light_spot.cpp.md) · [`blender_ssao.cpp`](blender_ssao.cpp.md) · [`dx11RainBlender.cpp`](dx11RainBlender.cpp.md)
**Tier floor** — T2: a decision tree that emits pass descriptions. What stops T3 is nothing intrinsic; it is grouped with T1 code only because the names it emits must match files on disk.

## Purpose

Deferred shading splits lighting from surface: the g-buffer holds position, normal and material index per pixel, and each light is then a full-screen-ish pass that reads those and adds its contribution to an accumulator target. This file holds the template for the one light that is always present and never local — the sun.

The sun is special in three ways that the element index encodes: its shadow map is *cascaded* (split by distance into a near, a middle and a far slice, each with its own projection), it contributes a screen-wide luminance estimate used by the tone-mapping stage, and it is the only light whose in-scattering ("god rays") is worth a separate volumetric pass.

## The element namespace

```text
ENUM SunElement
  SUN_NEAR        = 0   # nearest cascade; depth-clipped
  SUN_MIDDLE      = 1   # middle cascade; same emission as NEAR
  SUN_FAR         = 2   # farthest cascade; clipped by stencil only
  SUN_LUMINANCE   = 3   # screen luminance estimate, no accumulation
  SUN_NEAR_MINMAX = 4   # NEAR, with the min/max shadow-map acceleration texture
  SUN_RAIN_SMAP   = 5   # the rain shadow map (used by the rain templates, not here)
```

**Invariants** — `SUN_NEAR` and `SUN_MIDDLE` emit the *same* pass. They are separate indices so that the draw stage can bind a different cascade projection between them, not because the material differs.

## The shared inputs of every sun pass

```text
s_position     <- the g-buffer position target, point-sampled
s_normal       <- the g-buffer normal target, point-sampled
s_material     <- the material lookup volume, clamped/wrapped per its own convention
s_accumulator  <- the light accumulator, read back so the pass can add to it
s_lmap         <- the sun mask texture ("sunmask"), a fixed named texture
s_smap         <- the sun's shadow map
jitter0..3     <- four small dither textures, point-sampled, wrapped
```

**Invariants**

- The accumulator is bound as an *input* while also being the render target, which only works because the blend mode is additive and the shader reads the same pixel it writes. Where the device cannot blend the accumulator's format, the pass instead reads the accumulator explicitly and writes a replacement — that is what the `blend`/`dest` pair below selects.
- The four jitter textures are bound as a group, always four, always point-sampled and wrapped. They supply the per-pixel rotation of the shadow-map sampling pattern; using fewer, or filtering them, turns the soft-shadow noise into banding.

## `CBlender_accum_direct.compile(context)`

**Contract** — emits one sun accumulation pass for the requested cascade, or the luminance pass. No allocation, no I/O beyond shader-source resolution.

```text
FUNCTION compile(C)
  base_compile(C)

  blend = false                       # additive accumulation is disabled...
  dest  = blend ? ONE : ZERO
  IF sun_filter_enabled THEN          # ...and forced off again when the sun shadow
    blend = false ; dest = ZERO       #    is filtered in a later pass
  END

  SWITCH C.element
    CASE SUN_NEAR, SUN_MIDDLE:
      C.pass(vertex="accum_sun", pixel="accum_sun_near_<msaa><minmax>",
             fog=false, depth_test=true, depth_write=false,
             blend=blend, src=ONE, dst=dest)
      C.cull(none)
      C.depth(test=true, write=false, INVERTED)   # force reversed depth comparison
      bind_shared(C) ; bind_smap(C) ; bind_jitter(C)
      C.end()

    CASE SUN_FAR:
      # identical, minus the depth override: the far cascade covers everything
      # behind the near one and is clipped by stencil instead
      C.pass(vertex="accum_sun", pixel="accum_sun_far_<msaa>", ...)
      C.cull(none)
      bind_shared(C) ; bind_smap(C) ; bind_jitter(C)
      # the shadow map is sampled with a WHITE border, so anything outside the
      # cascade's footprint reads as fully lit rather than fully shadowed
      C.sampler_address("s_smap", BORDER) ; C.sampler_border("s_smap", opaque white)
      C.end()

    CASE SUN_LUMINANCE:
      C.pass(vertex="stub_notransform_aa_AA", pixel="accum_sun_<msaa>", fog=false,
             depth_test=false, depth_write=false)
      C.cull(none)
      bind s_position, s_normal, s_material, and s_smap <- generic scratch target 0
      bind_jitter(C)
      C.end()

    CASE SUN_NEAR_MINMAX:
      # as SUN_NEAR, plus the min/max shadow-map texture
      ... bind s_smap_minmax as well ...
```

**Invariants**

- The near and middle cascades **force an inverted depth comparison**. The pass is drawn as a volume (a box covering the cascade's world extent), and inverting the test turns "in front of the geometry" into "behind it", which is how the volume's far face is used to clip the cascade against the scene's depth without a separate stencil pass. The far cascade skips this because it has no far boundary to clip against.
- The far cascade's shadow map is addressed with a **white border**. Sampling outside a cascade means "this pixel is beyond the shadow map"; the correct answer there is *unshadowed*, and a border colour is the only way to say that without a branch per sample.
- Culling is disabled on every sun pass. The camera can be inside the cascade volume, in which case only its back faces are visible.

**Notes** — Two device-capability flags appear in the source and no longer discriminate: whether the device supports depth-format shadow maps and whether it does the percentage-closer filter in hardware. Older generations branched on them to pick a depth target or a colour target; every backend a rebuild would target has both, and the newer code paths read the depth target unconditionally. A rebuild implements the depth-target path only.

A third flag, "old shadow cascades", selects between drawing the pass as a *screen-space quad* and drawing it as a *volume*. The volume form is the current one and restricts the pass to the cascade's footprint; the quad form covers the whole screen and relies on the shader to reject pixels. Keep the volume form.

## `CBlender_accum_direct_msaa.compile(context)`

**Contract** — the same passes, compiled for one specific multisample *sample index*. The blender carries a name and a definition string; the definition parses as an integer sample index, which is stashed on the renderer for the duration of the compile so that the shader compiler bakes that index into the program. A blender constructed without a name compiles the sample-index-agnostic form.

**Invariants** — the sample index must be restored to "none" when the compile returns, on every path including the ones that emit nothing. The renderer-wide field it writes is effectively a parameter smuggled past the recorder's interface; a rebuild should pass it as a parameter and delete the save/restore.

**Notes** — This is the shape every `*_msaa` blender in this directory takes, and the rest are not described again: same elements, same bindings, the pixel program named with an `_msaa` suffix instead of `_nomsaa`, and the sample index set and cleared around the body. The reason a *separate blender object per sample* exists at all is that per-sample shading needs a program compiled for a literal sample index — the index cannot be a uniform — so the renderer instantiates one blender per sample count and compiles each.

## `CBlender_accum_direct_volumetric_msaa.compile(context)`

**Contract** — one pass that accumulates the sun's in-scattering through the light's own volume. Reads the material's first texture as the light's projected mask, the sun shadow map, and a fixed noise texture `fx/fx_noise` that animates the scattering. Emits nothing for any element but the first.

## `CBlender_accum_direct_volumetric_sun_msaa.compile(context)`

**Contract** — the screen-wide variant of the same: additive one-to-one blending with no depth test at all, reading only the shadow map and the g-buffer position. This is the pass that produces visible shafts across the whole frame rather than inside one light's volume.

**Notes** — The two volumetric templates name the *same* pixel program. They differ only in the geometry they are drawn with and in their blend and depth state, which is the whole point: the shader integrates along a ray, and what bounds the ray is the geometry the caller draws.
