# src/xrEngine/Environment.h

> Declares the weather system: a time-of-day frame, the blend of two frames, a local override volume, an ambient sound-and-effect set, and the environment that owns them all.

**Needs** — [`Include/xrRender/EnvironmentRender.h`](../Include/xrRender/EnvironmentRender.h.md) · [`xrSound/Sound.h`](../xrSound/Sound.h.md) · [`editor_base.h`](editor_base.h.md) · [`perlin.h`](perlin.h.md)
**Used by** — [`engine.hpp`](../Include/editor/engine.hpp.md) · [`EnvironmentRender.h`](../Include/xrRender/EnvironmentRender.h.md) · [`Blender_Recorder_StandartBinding.cpp`](../Layers/xrRender/Blender_Recorder_StandartBinding.cpp.md) · [`DetailManager.cpp`](../Layers/xrRender/DetailManager.cpp.md) · [`FTreeVisual.cpp`](../Layers/xrRender/FTreeVisual.cpp.md) · [`LightTrack.cpp`](../Layers/xrRender/LightTrack.cpp.md) · [`Light_DB.cpp`](../Layers/xrRender/Light_DB.cpp.md) · [`dxEnvironmentRender.cpp`](../Layers/xrRender/dxEnvironmentRender.cpp.md) · [`r__dsgraph_render.cpp`](../Layers/xrRender/r__dsgraph_render.cpp.md) · [`r__dsgraph_render_lods.cpp`](../Layers/xrRender/r__dsgraph_render_lods.cpp.md) · [`r__sector_traversal.cpp`](../Layers/xrRender/r__sector_traversal.cpp.md) · [`dx11DetailManager_VS.cpp`](../Layers/xrRenderDX11/dx11DetailManager_VS.cpp.md) · [`r2_rendertarget_phase_bloom.cpp`](../Layers/xrRender_R2/r2_rendertarget_phase_bloom.cpp.md) · [`editor_environment_ambients_ambient.hpp`](../editors/xrWeatherEngine/editor_environment_ambients_ambient.hpp.md) · _and 21 more_
**Tier floor** — T2: records and interfaces; the rendering half is behind a factory pointer.

## Purpose

Declares the surface implemented across [`Environment.cpp`](Environment.cpp.md) (the clock and frame selection), [`Environment_misc.cpp`](Environment_misc.cpp.md) (loading, blending, the sun model), [`Environment_render.cpp`](Environment_render.cpp.md) (draw ordering) and [`Environment_editor.cpp`](Environment_editor.cpp.md) (the authoring tool).

The one constant it defines is the length of a day in seconds, 86400. Every time value in the weather system is seconds since midnight against that.

## Exported units

- **`EnvModifier`** — a sphere placed in the level that biases the environment near it: far plane, fog colour and density, ambient, sky and hemisphere colour, each independently enabled by a flag. Loaded from a per-level binary file. See [`Environment_misc.cpp`](Environment_misc.cpp.md).
- **`EnvAmbient`** — the sound and particle life of a place: a set of sound channels each with a distance range and four period bounds, and a set of timed particle effects each optionally carrying a wind blast. Chosen per frame by the blender.
- **`EnvFrame`** (the environment descriptor) — one authored time-of-day frame. Its name *is* its timestamp, spelled `HH:MM:SS`. Carries sky and cloud textures and rotations, colours for sky, fog, rain, ambient, hemisphere and sun, far plane and fog distance and density, rain density, thunderbolt period and duration, wind speed and direction, sun direction or the azimuth that generates it, sun-shaft and water intensity, the tree-sway parameters, and references to a lens-flare definition, a thunderbolt collection and an ambient set.
- **`EnvFrameMixer`** — an `EnvFrame` that is the blend of two others, plus the blend weight, the modifier attenuation, the derived near and far fog planes, and the packed environment colour the renderer consumes. Also carries the flag that selects between the two generations' environment-colour conventions.
- **`Environment`** — the clock, the cycle and effect tables, the modifier list, the ambient list, and the three weather visual effects (rain, lens flare, thunderbolt). It is also an editor tool, which is why the weather editor is a mode of the running game rather than a separate program.
- **`environment flags` and `visibility distance`** — two module-wide console-settable values. The visibility multiplier scales every frame's far plane and is the player's draw-distance setting.

**Notes** — Two frames carry both an *authored* time and a *current* time. They differ only while a weather effect is spliced into a cycle, where the effect's frames are re-stamped onto the running clock; see [`Environment.cpp`](Environment.cpp.md). Keeping both is what makes the splice reversible.

**Notes** — A frame's name is parsed as a timestamp, so the identifier is data rather than a label. A rebuild may store the seconds directly, but must still accept the `HH:MM:SS` spelling since it is the section name in shipped configuration.

**Notes** — The cloud dome's geometry is a tessellated hemisphere generated at construction rather than loaded, and is held here as vertices and indices because both the sky renderer and the cloud renderer index into it.

**Notes** — The two renderer backends are declared friends of the environment, which is the original's way of saying that the rendering half reads the whole frame state directly. In a rebuild the same relationship is a read-only view handed across the renderer interface.
