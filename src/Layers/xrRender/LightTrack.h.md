# src/Layers/xrRender/LightTrack.h

> Declares the per-object lighting estimate: twenty-six sky rays, a sun ray, a set of tracked lights, and the six-faced ambient cube they accumulate into.

**Needs** — [`Include/xrRender/RenderVisual.h`](../../Include/xrRender/RenderVisual.h.md) · [`light.h`](light.h.md) · [`xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md)
**Used by** — [`LightTrack.cpp`](LightTrack.cpp.md) · [`R_Backend_Runtime.cpp`](R_Backend_Runtime.cpp.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md)
**Tier floor** — T2: it is ray queries and running averages; nothing touches the device.

## Purpose

Declares the surface implemented in [`LightTrack.cpp`](LightTrack.cpp.md). Every renderable object in the world owns one of these: it is the renderer's answer to "how lit is this thing, right here, right now", amortized over hundreds of frames.

## The constants

```text
hemisphere_samples = 26     # fixed directions on the upper hemisphere
energy_increment   = 4.0    # per second, when a light is visible
energy_decrement   = 2.0    # per second, when it is not
```

**Invariants** — Light visibility rises twice as fast as it falls. Stepping into light should be immediate and stepping out should linger, because the former is what the player notices and the latter is what makes a flickering partial occlusion look stable rather than strobing. The two numbers are the whole of the temporal response and they are asymmetric on purpose.

## Exported units

- **`ObjectLighting`** — the record. Holds the tracked lights with their per-light running visibility, the sky-ray results with their ray caches, the ambient cube in raw and smoothed forms, the scalar sky and sun terms, the approximate average colour, and the shadow-allocation bookkeeping.
- **`Item`** — one tracked light: which light, its ray cache, its instantaneous test result and its running energy.
- **`Light`** — one light selected for this frame, with its energy-weighted colour.
- **`add(light)`** — note that a light reaches this object.
- **`update(object)` / `update_smooth(object)`** — recompute, and advance the temporal smoothing.
- **`luminance()` / `sky_term()` / `sun_term()` / `ambient_cube()` / `average_colour()`** — the read side, each of which lazily advances the smoothing if this frame has not yet.
- **`force_mode(mode)`** — which of the three estimates (sun, sky, lights) to compute for this object.

**Notes** — The ambient cube is six floats, one per axis direction, and the accessor hands out a raw pointer to them for a shader constant upload. That is the one place the record's layout is load-bearing outside the file.
