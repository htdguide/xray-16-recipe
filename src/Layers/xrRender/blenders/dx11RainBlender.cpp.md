# src/Layers/xrRender/blenders/dx11RainBlender.cpp

> The four-step screen-space wet-surface effect: decide what the rain reaches, perturb those surfaces' normals with animated water, and raise their gloss.

**Needs** — [`dx11RainBlender.h`](dx11RainBlender.h.md) · [`blender_light_direct.cpp`](blender_light_direct.cpp.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md) · [`dxRainRender.cpp`](../dxRainRender.cpp.md)
**Used by** — [`dx11RainBlender.h`](dx11RainBlender.h.md) · [`glRainBlender.cpp`](glRainBlender.cpp.md)
**Tier floor** — T2: a decision tree that emits pass descriptions, with fixed texture paths into the shipped data.

## Purpose

Rain in this engine is two separate things. The falling streaks and the splash sprites are geometry, and they live in [`dxRainRender.cpp`](../dxRainRender.cpp.md). This file is the other half: the *wetness* of the world, applied as a screen-space post-process over the finished g-buffer.

The idea is to reuse the sun's shadow machinery for a light that comes from straight above. The renderer renders a shadow map from the rain's direction; a pixel that is unoccluded in that map is a pixel the rain reaches, and therefore a pixel that should be wet. Making it wet means two edits to the g-buffer: replace its normal with a rippled one, and raise its specular gloss.

## The element namespace

```text
ENUM RainElement                # non-multisampled template
  TEST          = 0   # a visualization/diagnostic layer over the accumulator
  PATCH_NORMAL  = 1   # compute the wet normal into the accumulator (used as scratch)
  APPLY_NORMAL  = 2   # write that normal back into the g-buffer normal target
  APPLY_GLOSS   = 3   # multiply the g-buffer's gloss channel down or up
```

The multisampled template drops `TEST` and renumbers the remaining three from zero.

## The shadow-map size switch

**Contract** — every compile in this file begins by setting the renderer's current shadow-map size to the *rain* shadow-map size, and ends by restoring it to the sun's. The size is baked into the compiled shader as a constant (the filter kernel's texel step depends on it), so it must be correct while the pass is compiled, not while it is drawn.

**Invariants** — the restore must happen on every exit path. This is the same smuggled-parameter pattern as the multisample sample index, and a rebuild should pass both as arguments to the compile instead of writing renderer-wide state.

## `CBlender_rain.compile(context)`

**Contract** — emits one full-screen pass per element. All four force an inverted depth comparison, are depth-tested but never depth-writing, and bind the rain shadow map in place of the sun's.

```text
FUNCTION compile(C)
  base_compile(C)
  smap_size = rain_smap_size

  common_bindings(C):
    s_position, s_material, s_lmap <- the sun mask texture,
    s_smap <- the RAIN shadow depth target, and the four jitter textures

  SWITCH C.element
    CASE TEST:
      C.pass(v="stub_notransform_2uv", p="rain_layer", depth_test=true, blend=false)
      C.depth(test=true, write=false, INVERTED)
      common_bindings + s_normal + s_accumulator
      C.texture("s_water", "water/water_normal")
      C.end()

    CASE PATCH_NORMAL:
      C.pass(v="stub_notransform_2uv", p="rain_patch_normal_<msaa>", depth_test=true)
      C.depth(test=true, write=false, INVERTED)
      common_bindings + s_normal + s_diffuse <- the g-buffer albedo target
      C.texture("s_water",     "water/water_SBumpVolume")     # the ripple volume
      C.texture("s_waterFall", "water/water_flowing_nmap")     # the running-down normal map
      C.end()

    CASE APPLY_NORMAL:
      C.pass(v="stub_notransform_2uv", p="rain_apply_normal_<msaa>", depth_test=true)
      C.depth(test=true, write=false, INVERTED)
      common_bindings                                    # note: s_normal NOT bound
      C.texture("s_patched_normal", the accumulator)     # what PATCH_NORMAL wrote
      IF gbuffer_packs_normal_in_two_channels
        THEN C.color_write(r, g)
        ELSE C.color_write(r, g, b)
      C.end()

    CASE APPLY_GLOSS:
      C.pass(v="stub_notransform_2uv", p="rain_apply_gloss_<msaa>", depth_test=true,
             blend=true, src=ONE, dst=ONE)
      C.depth(test=true, write=false, INVERTED)
      common_bindings
      C.texture("s_patched_normal", the accumulator)
      C.blend_factors(src=ZERO, dst=SRC_COLOR)     # override: a MULTIPLY, not an add
      C.end()

  smap_size = sun_smap_size
```

**Invariants**

- The **accumulator is used as scratch** between `PATCH_NORMAL` and `APPLY_NORMAL`. That is only safe because this effect runs before any light has accumulated into it, and it is the reason the three passes must run in this order with nothing between them.
- `APPLY_NORMAL` masks colour writes to exactly the channels the g-buffer's normal occupies: two when the renderer packs the normal into two channels and reconstructs the third, three otherwise. Writing the fourth channel would destroy the gloss the very next pass is about to modify.
- `APPLY_GLOSS` opens with an additive blend and then *overwrites* the blend factors with `ZERO`/`SOURCE_COLOR`, making the pass a multiply. The pass declaration's factors are discarded; only the override survives. A rebuild should declare the multiply directly — but must reproduce the multiply, not the add.
- The three water textures are referenced by fixed paths in the shipped data: `water/water_normal`, `water/water_SBumpVolume`, `water/water_flowing_nmap`. They are not material parameters and not render targets, and they must exist in the mounted filesystem.

**Notes** — The first element, `TEST`, is a leftover development layer: it names a pixel program that shades the rain coverage directly over the accumulator, and the multisampled template does not have it. A rebuild may omit it; the effect is the other three.

`CBlender_rain_msaa` is the same three working elements with `_msaa`-suffixed programs and a baked sample index, described once in [`blender_light_direct.cpp`](blender_light_direct.cpp.md). It sets both the sample index *and* the rain shadow-map size, and restores both.
