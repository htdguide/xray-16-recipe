# src/Layers/xrRender/Light_DB.cpp

> The level's lights: read the baked set, pick the sun out of it, and every frame put the sun where the weather says the sun is — five hundred metres behind the camera.

**Needs** — [`Light_DB.h`](Light_DB.h.md) · [`light.h`](light.h.md) · [`Light_Package.h`](Light_Package.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`Common/LevelStructure.hpp`](../../Common/LevelStructure.hpp.md)
**Used by** — reached through its declarations in [`Light_DB.h`](Light_DB.h.md); callers name that, not this file.
**Tier floor** — T1: two different frozen light records are read as byte images, and the count is derived by dividing the chunk length by the record size.

## Purpose

Every light in a level comes from one of three places: baked into the level file by the level compiler, created at run time by the game (a flashlight, a muzzle flash), or *the sun*, which is one light and is entirely driven by the weather. This file owns the first and the third.

## The baked light records — frozen, and two different formats

```text
The level file's dynamic-light chunk is a flat array with NO count. The count
is (chunk length) / (4 + record size), and a chunk whose length is not an
exact multiple is a corrupt level.

Each entry is a 4-byte controller identifier that is SKIPPED, followed by:

RECORD BakedLight
  type          : point | spot | directional
  diffuse,
  specular,
  ambient       : colour
  position      : point
  direction     : vector3
  range         : real
  falloff, attenuation constants, cone angles
```

The hemisphere set comes from a *different* file — the level compiler's own output, not the shipped level data — with its own record layout, and is read only when that file is present.

**Invariants**

- **The count-by-division is the whole format.** A rebuild must reproduce the record's exact byte size or it reads garbage, and the assertion that the division is exact is the only integrity check there is.
- **The four-byte controller identifier is skipped and never read.** It named an animation controller in a system that no longer exists. It must still be skipped, because it is in the stride.
- **Specular is not read from the file.** It is overwritten on load with the diffuse colour at one fifth strength. The authored specular value in the shipped levels is ignored entirely. There is no comment; the effect is a uniform, mild specular response from every baked light, which is a global art decision made in code.
- **Exactly one directional light is expected, and it is the sun.** The load asserts its presence — a level without one refuses to load. Every other baked light is a point light, and spot lights never appear in the baked set despite the format supporting them.
- Baked point lights are **shadow-casting on the newer renderers and not on the oldest**. The oldest bakes their contribution into light maps and would double-count.

## `update()` — the sun, every frame

**Contract** — places and colours the sun from the weather system's current keyframe, then empties this frame's light package. Called once per frame before visibility.

```text
FUNCTION update()
  environment = the weather system's current interpolated keyframe
  REQUIRE environment.sun_direction points DOWNWARD

  IF the renderer uses a moving sun
    direction = normalize(environment.sun_direction)
  ELSE
    direction = normalize(environment.sun_direction + (0, -0.75, 0))   # see below

  position = camera position - direction * 500
  sun.rotation = direction, keeping the sun's existing right vector
  sun.colour   = environment.sun_colour * a global luminance scale
  sun.range    = 600

  package.clear()
```

**Invariants**

- **The sun is a point light five hundred metres behind the camera with a six-hundred-metre range, and it follows the camera.** It is not treated as a direction. That is what lets the same light machinery — attenuation, shadow allocation, the occlusion query — handle it, and the range is chosen so that the whole potentially-visible world falls inside it. A rebuild with a genuine directional light type does not need this and should not reproduce it; what it must reproduce is that *shadow cascades are built around the camera, not around the world*.
- **The downward-pointing assertion is load-bearing.** A weather configuration whose sun direction has a non-negative vertical component means the sun is below the horizon, and the whole shadow and cascade setup divides by assumptions that fail there. The checked build logs the entire keyframe before aborting, which says how often this was hit during authoring.
- **The static-sun path biases the direction downward by 0.75 and renormalizes.** The comment says only that the configured direction "can point up". The loop that keeps adding the configured direction until the sum has non-trivial magnitude is defensive against the degenerate case where the bias exactly cancels it, and gives up after ten tries. This is a compatibility shim for one of the shipped games' weather configurations, not a design; a rebuild should fix the data.
- The colour is multiplied by a **global luminance scale** that is a console setting. It is the one knob that changes the overall exposure of an outdoor scene, and it is applied here rather than in the tone mapping.

## `add_light(light)`

**Contract** — submits a light to this frame's package, at most once per frame per light. Idempotent within a frame by a stamped frame number.

**Invariants**

- **A baked static light is dropped entirely unless a setting asks for it.** The newer renderers bake nothing and want them all; the oldest has them in its light maps and must not add them again. The setting that re-enables them on the newer path exists so a level can be viewed with the old lighting for comparison.
- Shadowing is force-disabled on every light when a global no-shadows setting is on. That is the cheapest quality reduction the renderer has.
- The frame stamp is what makes this safe to call from several places — an object may reach the same light through several sectors.

## `create()`

**Contract** — a fresh light: not static, not active, shadow-casting. The defaults are the *dynamic* case, because everything that calls this outside the load path is creating a dynamic light.

## `load_hemisphere()`

**Contract** — reads a second, optional light set from the level compiler's own output file, used only for the dynamic ambient-occlusion estimate in [`LightTrack.cpp`](LightTrack.cpp.md). Only point lights are taken; the records' attenuation parameters are preserved, unlike the baked set's.

**Notes** — This set exists because the baked lights the *renderer* uses have had their attenuation normalized away, while the ambient-occlusion estimate needs the compiler's original falloff to weight a light's contribution to a surface it cannot trace to. It is the one place in the engine that reads a level-compiler intermediate file at run time, and its absence is not an error — the estimate simply has fewer lights to work with.
