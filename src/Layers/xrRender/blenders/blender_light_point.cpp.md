# src/Layers/xrRender/blenders/blender_light_point.cpp

> The template that adds one omnidirectional light's contribution to the accumulator, in five flavours of increasing cost.

**Needs** — [`blender_light_point.h`](blender_light_point.h.md) · [`blender_light_direct.cpp`](blender_light_direct.cpp.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md)
**Used by** — [`blender_light_point.h`](blender_light_point.h.md) · [`blender_light_spot.cpp`](blender_light_spot.cpp.md)
**Tier floor** — T2: a decision tree that emits pass descriptions.

## Purpose

A point light in a deferred renderer is a *volume* drawn into the accumulator: a sphere-ish hull covering the light's reach, whose pixel program reads the g-buffer and adds a contribution. This template emits that pass. Its five elements are not five different lights — they are five quality/feature levels that the renderer picks between per light per frame, based on how much the light matters this frame and whether it casts a shadow at all.

## The element namespace

```text
ENUM LightElement            # shared with the spot template
  L_FILL       = 0   # fill the light's projective mask target from its texture
  L_UNSHADOWED = 1   # no shadow map; cheapest
  L_NORMAL     = 2   # shadowed, shadow map allocated at a reduced size
  L_FULLSIZE   = 3   # shadowed, shadow map at full size
  L_TRANSLUENT = 4   # shadowed, and the shadow carries colour through translucent occluders
```

**Invariants** — `L_NORMAL` and `L_FULLSIZE` emit an *identical* pass for a point light; only the shadow map's allocated resolution differs, and that is the caller's decision, not the material's. (For a spot light they differ, and there the two elements name different programs — see [`blender_light_spot.cpp`](blender_light_spot.cpp.md).)

## `CBlender_accum_point.compile(context)`

**Contract** — emits one accumulation pass for the requested element. No depth test on any of them: the light volume is clipped by stencil, already written by the mask template.

```text
FUNCTION compile(C)
  base_compile(C)
  blend = device_can_blend_accumulator_format
  dest  = blend ? ONE : ZERO

  SWITCH C.element
    CASE L_FILL:
      # a plain copy of the light's own texture into the mask target
      C.pass(vertex="stub_notransform", pixel="copy", depth_test=false, depth_write=false)
      C.texture("s_base", C.textures[0]) ; C.end()

    CASE L_UNSHADOWED:
      C.pass(vertex="accum_volume", pixel="accum_omni_unshadowed",
             depth_test=false, depth_write=false, blend=blend, src=ONE, dst=dest)
      bind s_position, s_normal, s_material, s_lmap <- C.textures[0], s_accumulator
      C.end()

    CASE L_NORMAL, L_FULLSIZE:
      C.pass(vertex="accum_volume", pixel="accum_omni_normal", ... same blend ...)
      bind the same five, plus s_smap <- the sun/spot shadow depth target
      bind_jitter(C)
      C.end()

    CASE L_TRANSLUENT:
      C.pass(vertex="accum_volume", pixel="accum_omni_transluent", ... same blend ...)
      # the difference: s_lmap is bound to the shadow-map COLOUR surface, not to
      # the light's own texture, so what passes through the occluder is tinted
      bind s_position, s_normal, s_material, s_lmap <- shadow colour surface,
         s_smap <- shadow depth, s_accumulator
      bind_jitter(C)
      C.end()
```

**Invariants**

- The light's own texture in slot 0 is the omni light's *projected colour* — a cube or a sphere-mapped attenuation texture. Under `L_TRANSLUENT` that slot is reused for the shadow map's colour surface, which is why the two cannot be merged.
- The accumulator is bound as an input and is also the target, exactly as for the sun; `blend` decides whether the pass adds via the blender or re-reads and replaces.

**Notes** — The `L_FILL` element is not lighting at all: it exists because a light with a projected texture needs that texture resident in a render target of the right size before the accumulation pass samples it, and a blit is the cheapest way to get it there. The vertex program names differ across backends (`null` versus `stub_notransform`) solely because one device supplies a pass-through vertex stage implicitly and the other does not.

The sample-indexed variant, `CBlender_accum_point_msaa`, is the same body with `_msaa`-suffixed pixel programs and the sample index stashed and restored around it; the mechanism is described once in [`blender_light_direct.cpp`](blender_light_direct.cpp.md).
