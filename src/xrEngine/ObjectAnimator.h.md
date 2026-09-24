# src/xrEngine/ObjectAnimator.h

> Declares the rigid-object animator: a bank of named transform motions with one playing at a time.

**Needs** — [`ObjectAnimator.cpp`](ObjectAnimator.cpp.md) · [`xrCore/Animation/Motion.hpp`](../xrCore/Animation/Motion.hpp.md)
**Used by** — [`ObjectAnimator.cpp`](ObjectAnimator.cpp.md) · [`ActorEffector.cpp`](../xrGame/ActorEffector.cpp.md) · [`ActorEffector_script.cpp`](../xrGame/ActorEffector_script.cpp.md) · [`TorridZone.cpp`](../xrGame/TorridZone.cpp.md) · [`script_particles.cpp`](../xrGame/script_particles.cpp.md) · [`script_particles.h`](../xrGame/script_particles.h.md)
**Tier floor** — T2: evaluates a curve per frame and builds a transform

## Purpose

Declares the surface implemented in [`ObjectAnimator.cpp`](ObjectAnimator.cpp.md).

Exported units:

- `CObjectAnimator` — owns a sorted bank of named object motions loaded from one file,
  plays one at a time at an adjustable speed with optional looping, and exposes the
  world transform that the currently playing motion evaluates to. Also `Clear`, `Load`,
  `Play`, `Stop`, `Pause`, `IsPlaying`, `GetLength`, `Update`, and an editor-only
  `DrawPath` that renders the motion's translation curve.
