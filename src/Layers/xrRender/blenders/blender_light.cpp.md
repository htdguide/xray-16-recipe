# src/Layers/xrRender/blenders/blender_light.cpp

> The forward renderer's additive light template: the fixed-function two-stage stack that multiplies a 2D attenuation map by a 1D falloff and by the light's colour.

**Needs** — [`blender_light.h`](blender_light.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md)
**Used by** — [`blender_light.h`](blender_light.h.md)
**Tier floor** — T1: it describes a fixed-function texture-combiner stack, a device model that predates programmable shading and must be emulated where the device no longer has one.

## Purpose

The template registered under the class identifier `"LIGHT   "`, and the only file in this directory that belongs to the *oldest* renderer generation — the one that lights forward, one additive pass per light, with no shaders at all. It exists in the shipped material library and must load there.

Where the deferred templates describe a program and its inputs, this one describes a **combiner stack**: an ordered list of texture stages, each of which takes a colour and an alpha from a source, applies one operation, and hands the result to the next. Reproducing the stack faithfully is the whole contract; on a device with no fixed-function pipeline it must be emulated by a small shader that performs the same two modulations.

## `CBlender_LIGHT.compile(context)`

**Contract** — emits one additive pass with two texture stages. No parameters, no elements: the same pass whatever the element index.

```text
FUNCTION compile(C)
  base_compile(C)
  C.pass_begin()
    C.depth(test=true, write=false)          # a light adds to what is there
    C.blend(on, src=ONE, dst=ONE, alpha_test=false, alpha_ref=0)
    C.lighting(off) ; C.fog(off)

    # stage 0 — the 2D attenuation map, modulated by the light colour
    C.stage_begin()
      C.stage_address(CLAMP)
      C.stage_color(TEXTURE, MODULATE, CONSTANT_FACTOR)
      C.stage_alpha(TEXTURE, MODULATE, CONSTANT_FACTOR)
      C.stage_texture("$base0") ; C.stage_matrix("$null", 0) ; C.stage_constant("$null")
    C.stage_end()

    # stage 1 — the 1D falloff along the light's axis, modulated by stage 0's result
    C.stage_begin()
      C.stage_address(CLAMP)
      C.stage_color(TEXTURE, MODULATE, CURRENT)
      C.stage_alpha(TEXTURE, MODULATE, CURRENT)
      C.stage_texture("$base1") ; C.stage_matrix("$null", 1) ; C.stage_constant("$null")
    C.stage_end()
  C.pass_end()
```

**Invariants**

- Both stages **clamp**. The attenuation textures encode "this is how bright the light is at this distance"; wrapping either one tiles the light's falloff and lights the world in a grid.
- Depth is tested but not written. An additive light pass must not disturb the depth buffer the opaque pass established, or the next light will be occluded by it.
- The two texture slots are positional references — `$base0` and `$base1` — into the material instance's texture list, and the two matrix slots are the per-stage texture transforms that project world position into the attenuation maps. `$null` means "no animated transform"; the light's projection matrix is supplied by the draw code, not by the material. The positional-name convention is documented in [`Blender_Recorder.cpp`](../Blender_Recorder.cpp.md).
- The light's colour arrives through the device's single constant colour factor, which is why both stages modulate against it rather than against a per-vertex colour: this pass is issued for geometry whose vertex colours already mean something else.

**Notes** — The file refuses to compile against any renderer generation but the first. That is a build-time assertion in the original and a statement of fact in a rebuild: nothing else uses a combiner stack, and the deferred path lights the same scene through [`blender_light_point.cpp`](blender_light_point.cpp.md) and [`blender_light_spot.cpp`](blender_light_spot.cpp.md) instead.
