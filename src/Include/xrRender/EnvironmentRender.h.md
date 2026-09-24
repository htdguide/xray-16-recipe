# src/Include/xrRender/EnvironmentRender.h

> The renderer's half of the weather system: the sky and cloud domes, the per-keyframe visual resources, and the blend between two keyframes.

**Needs** — [`RenderFactory.h`](RenderFactory.h.md) · [`particles_systems_library_interface.hpp`](particles_systems_library_interface.hpp.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`RenderFactory.h`](RenderFactory.h.md) · [`particles_systems_library_interface.hpp`](particles_systems_library_interface.hpp.md) · [`dxEnvironmentRender.cpp`](../../Layers/xrRender/dxEnvironmentRender.cpp.md) · [`dxEnvironmentRender.h`](../../Layers/xrRender/dxEnvironmentRender.h.md) · [`Environment.h`](../../xrEngine/Environment.h.md)
**Tier floor** — T2: two small interfaces over device resources; the caller never sees a buffer or a texture.

## Purpose

The engine owns the weather model: a list of named weather cycles, each a sequence of keyframes at times of day, each keyframe carrying a fog colour and density, an ambient and hemisphere colour, a sun colour and direction, sky and cloud texture names, wind, thunder and ambient-sound settings. It interpolates between the two keyframes bracketing the current game time, every frame, and publishes the result.

None of that needs a graphics device. What *does* is drawing the sky: two textured domes — a sky dome and a cloud layer — blended between the two keyframes' textures, plus the resources each keyframe owns. That split is exactly this file: the engine keeps the weather state, the renderer keeps the domes and the textures.

## State

Both interfaces are owned through [`FactoryPtr.h`](FactoryPtr.h.md): the weather manager holds one environment renderer, each keyframe holds one descriptor renderer.

```text
RECORD EnvironmentRenderState
  sky_dome    : Model          # a textured hemisphere, drawn behind everything
  cloud_dome  : Model          # a second layer, scrolled by wind
  current     : resolved blend of two keyframes' sky and cloud materials

RECORD EnvDescriptorRenderState
  sky_material    : Material   # from the keyframe's authored texture name
  cloud_material  : Material
```

## `IEnvDescriptorRender` — the per-keyframe half

**Contract** — one instance per authored weather keyframe, created when the keyframe is parsed and destroyed with it. It resolves the keyframe's sky and cloud texture names into materials on device creation and releases them on device destruction. It draws nothing itself; the environment renderer reads its resources when blending.

```text
FUNCTION on_device_create(keyframe)     # resolve this keyframe's materials
FUNCTION on_device_destroy()            # release them
FUNCTION copy(other)                    # the FactoryPtr duplication hook
```

**Invariants** — created and destroyed in lockstep with the whole weather set: the manager walks every cycle and every effect and calls these on each keyframe, so a keyframe never holds device resources while the device is gone.

## `IEnvironmentRender` — the drawing half

```text
FUNCTION on_device_create()             # build the two domes
FUNCTION on_device_destroy()            # release them
FUNCTION clear()                        # drop the current blend
FUNCTION copy(other)
FUNCTION blend(current_state, from : EnvDescriptorRender, to : EnvDescriptorRender)
FUNCTION render_sky(environment)
FUNCTION render_clouds(environment)
FUNCTION particles_library() -> ParticlesLibrary
```

### `blend`

**Contract** — called once per frame, immediately after the engine has interpolated its own weather values, with the two bracketing keyframes' renderer halves and the freshly mixed engine-side state. The implementor arranges for the sky and cloud passes to sample both keyframes' textures and cross-fade between them by the same weight the engine used.

**Invariants** — the engine's interpolation and this call must use the same weight, or the sky's colour and its texture will disagree. The engine computes the weight from game time against the two keyframes' scheduled times and does not pass it here — the implementor reads it from the mixed state. That coupling is implicit and is the one thing in the weather system most likely to be got wrong in a rebuild.

### `render_sky` / `render_clouds`

**Contract** — draw the two domes, centred on the camera, with depth writes off. Two calls rather than one because they are issued at different points in the frame: the sky goes down before the world, and the clouds after the world's opaque pass so they sort against distant geometry correctly.

### `particles_library`

**Contract** — hands back the renderer's catalogue of particle-effect definitions, for enumeration only.

**Notes** — This method is on the wrong interface and it is worth saying so plainly: a catalogue of particle definitions has nothing to do with weather. It is here because the *weather editor* needs to offer the author a list of particle effect names for thunder and rain, and the environment renderer was the editor's existing handle into the renderer. A rebuild should expose the catalogue from wherever particle definitions are loaded. See [`particles_systems_library_interface.hpp`](particles_systems_library_interface.hpp.md).

## Notes

There is no create/destroy pair for the domes on the weather side because the device create and destroy calls serve as both — the domes are built from a fixed procedural mesh, not from content, so they can be rebuilt at any time. A device reset therefore costs the sky nothing but a rebuild of two small meshes.
