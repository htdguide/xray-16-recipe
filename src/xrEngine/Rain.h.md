# src/xrEngine/Rain.h

> Declares the rain effect: a camera-following drop field, a splash-particle pool and the ambient bed.

**Needs** — [`Rain.cpp`](Rain.cpp.md) · [`Environment.h`](Environment.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`RainRender.h`](../Include/xrRender/RainRender.h.md) · [`dxRainRender.cpp`](../Layers/xrRender/dxRainRender.cpp.md) · [`Environment.cpp`](Environment.cpp.md) · [`Environment_misc.cpp`](Environment_misc.cpp.md) · [`Environment_render.cpp`](Environment_render.cpp.md) · [`Rain.cpp`](Rain.cpp.md)
**Tier floor** — T1: a fixed-size particle pool threaded as two intrusive lists, sized so the effect never allocates during a frame

## Purpose

Declares the surface implemented in [`Rain.cpp`](Rain.cpp.md). The split between this file
and the renderer's rain drawing is deliberate: this side owns where drops *are* and when
they hit, the renderer side owns how they look, and the two agree only on the drop record.
The renderer half is named as a friend so it can read the drop field directly without a
copy — an artefact of C++ access control, not a design decision; a rebuild exposes the
field as a read-only view.

Exported units:

- `CEffect_Rain` — the effect. Constructed once per game session, not per level.
- `OnFrame` — advances the ambient sound and the on/off state from the current weather.
- `Render` — hands the whole effect to the renderer's rain drawer.

Internal to the effect, but part of its contract with the renderer:

- `Item` — one falling drop: current origin, predicted impact point, direction, speed,
  the times it expires and hits, and which of two texture variants it uses.
- `Particle` — one splash, a transform and a bounding sphere with a countdown, held in a
  fixed pool threaded as an active list and an idle list.
