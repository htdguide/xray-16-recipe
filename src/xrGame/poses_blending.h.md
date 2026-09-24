# src/xrGame/poses_blending.h

> Declares the two-pose blend: interpolation between a start and an end transform, and the timed run between them. Implemented in [`poses_blending.cpp`](poses_blending.cpp.md).

**Needs** — [`poses_blending.cpp`](poses_blending.cpp.md)
**Used by** — [`animation_movement_controller.cpp`](animation_movement_controller.cpp.md) · [`animation_movement_controller.h`](animation_movement_controller.h.md) · [`poses_blending.cpp`](poses_blending.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Two small types, separated because they answer different questions. The first is *where
between these two poses is a given fraction*; the second adds *how long the trip takes*, so
a caller holds a clock rather than a fraction.

Splitting them means the interpolation is reusable by anything with its own notion of
progress, while the blend is the ready-made "move from here to there over this long".

Exported units:

- `poses_interpolation` — construct from two transforms; ask for the pose at a fraction.
- `poses_blending` — the same plus a duration; ask for the pose at a time, and ask whether
  the time has run out.
