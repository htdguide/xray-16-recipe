# src/xrEngine/FDemoPlay.h

> Declares the demo-playback camera effector.

**Needs** — [`Effector.h`](Effector.h.md) · [`xrCore/Animation/Motion.hpp`](../xrCore/Animation/Motion.hpp.md)
**Used by** — [`FDemoPlay.cpp`](FDemoPlay.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`FDemoPlay.cpp`](FDemoPlay.cpp.md).

## Exported units

- **`DemoPlayback`** — a camera effector holding a recorded path, a playback cursor, a cycle count and a frame-time sample table. Constructed with the path name, the seconds per keyframe, a cycle count where zero means once, and a lifetime defaulting to one hour. See the implementation twin.
- **`apply`** — the per-frame step.

**Notes** — The one-hour default lifetime is an upper bound on any demo, not a meaningful duration. A keyframe demo ends by exhausting its cycles; the lifetime only catches an animation-curve demo whose curve never wraps.
