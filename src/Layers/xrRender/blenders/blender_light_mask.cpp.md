# src/Layers/xrRender/blenders/blender_light_mask.cpp

> The templates that write the stencil masks bounding each light, and that copy the accumulator between its temporary and real targets.

**Needs** — [`blender_light_mask.h`](blender_light_mask.h.md) · [`blender_light_direct.cpp`](blender_light_direct.cpp.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — [`blender_light_mask.h`](blender_light_mask.h.md)
**Tier floor** — T2: a decision tree that emits pass descriptions.

## Purpose

Two jobs share this file because both are "a pass that writes no colour and exists to prepare the next pass".

The first is **masking**: before a light's accumulation pass runs, its volume is rasterized once with colour writes disabled, so that the stencil buffer marks exactly the pixels the volume covers. The accumulation pass then costs only those pixels. For the sun there is no volume, so the mask is instead derived from the g-buffer normal — a pixel with no surface written has nothing to light.

The second is **accumulator shuffling**: the accumulator sometimes has to be written to a temporary target (because the device could not blend its format, or because a sun-filter pass needed a clean surface) and then copied back. Those copies are passes too, and they live here.

## The element namespace

```text
ENUM MaskElement
  MASK_SPOT      = 0   # stencil mask for a cone light's volume
  MASK_POINT     = 1   # stencil mask for an omni light's volume
  MASK_DIRECT    = 2   # stencil mask for the sun, derived from the g-buffer
  MASK_ACCUM_VOL = 3   # copy accumulator temp -> real, drawn as a volume
  MASK_ACCUM_2D  = 4   # copy accumulator temp -> real, drawn as a screen quad
  MASK_ALBEDO    = 5   # copy the real accumulator out, for the albedo combine
```

## `CBlender_accum_direct_mask.compile(context)`

**Contract** — emits one no-colour pass per element. Allocates nothing.

```text
FUNCTION compile(C)
  base_compile(C)
  SWITCH C.element
    CASE MASK_SPOT, MASK_POINT:
      # rasterize the light volume; depth-tested so the volume is clipped by scene depth
      C.pass(vertex="accum_mask", pixel=<do-nothing>, depth_test=true, depth_write=false)
      C.color_write(none)
      C.end()
      # NOTE: culling, the stencil operation and the stencil reference are set by the
      # draw code, not here — the same pass is issued twice with opposite cull and
      # opposite stencil op to mark the volume's interior.

    CASE MASK_DIRECT:
      # a screen quad; alpha-test with reference 1 against the g-buffer normal, so a
      # pixel with no surface fails and leaves the stencil unmarked
      C.pass(vertex="stub_notransform_t", pixel="accum_sun_mask_<msaa>",
             depth_test=false, depth_write=false,
             blend=true, src=ZERO, dst=ONE, alpha_test=true, alpha_ref=1)
      C.texture("s_normal", g-buffer normal) ; C.texture("s_position", g-buffer position)
      C.color_write(none)
      C.end()

    CASE MASK_ACCUM_VOL:
      C.pass(vertex="accum_volume", pixel="copy_p_<msaa>", depth_test=false)
      C.texture("s_generic", accumulator temporary) ; C.end()

    CASE MASK_ACCUM_2D, MASK_ALBEDO:
      C.pass(vertex="stub_notransform_t", pixel="copy_<msaa>", depth_test=false)
      C.texture("s_generic", MASK_ACCUM_2D ? accumulator temporary : accumulator)
      C.end()
```

**Invariants**

- The sun mask's blend is `ZERO * source + ONE * destination` — it writes nothing to colour by construction, *and* colour writes are disabled on top of that. The redundancy is deliberate on devices where the alpha test is only evaluated for pixels that reach the blender.
- The alpha reference of **1** is the whole trick of the sun mask: the g-buffer normal target's alpha is zero where no geometry was written and non-zero where it was, so "alpha ≥ 1" is exactly "a surface exists here".
- The two copy shapes are not interchangeable. `MASK_ACCUM_VOL` is drawn as the light's volume and copies only what that light touched; `MASK_ACCUM_2D` is a full-screen quad and copies everything. Using the quad where the volume was meant costs a full-screen copy per light.

**Notes** — The pixel program for the two light-volume masks is a do-nothing program. Only rasterization coverage matters. On devices that do not require a pixel stage, none is bound.

`CBlender_accum_direct_mask_msaa` is the same body with `_msaa`-suffixed copy programs and a baked sample index, with one asymmetry worth reproducing: `MASK_ALBEDO` uses the **non**-sample-indexed copy even in the sample-indexed blender, because the albedo combine reads a resolved surface.
