# src/xrEngine/EffectorPP.h

> A post-process effector: a timed contribution to the screen's colour grading, blur, grain and duality.

**Needs** — [`CameraDefs.h`](CameraDefs.h.md) · [`xrCore/PostProcess/PPInfo.hpp`](../xrCore/PostProcess/PPInfo.hpp.md)
**Used by** — [`CameraManager.cpp`](CameraManager.cpp.md) · [`EffectorPP.cpp`](EffectorPP.cpp.md) · [`ActorEffector.cpp`](../xrGame/ActorEffector.cpp.md) · [`CameraEffector.h`](../xrGame/CameraEffector.h.md) · [`PostprocessAnimator.cpp`](../xrGame/PostprocessAnimator.cpp.md) · [`PostprocessAnimator.h`](../xrGame/PostprocessAnimator.h.md) · [`SleepEffector.cpp`](../xrGame/SleepEffector.cpp.md) · [`SleepEffector.h`](../xrGame/SleepEffector.h.md)
**Tier floor** — T3: a countdown and a parameter contribution.

## Purpose

Declares the surface implemented in [`EffectorPP.cpp`](EffectorPP.cpp.md): the screen-effect counterpart of the camera effector. Radiation sickness, a concussion, drunkenness, a night-vision tint, the fade at a death — each is one of these, contributing a parameter set the camera manager composites.

## Exported units

- **`PPEffector`** — identity, lifetime, and two flags.
- **`apply`** — contribute a parameter set for this frame; see the implementation twin.
- **`is_valid`** — whether lifetime remains.
- **`stop`** — end the effect now. Takes a fade speed which the base implementation ignores, expiring immediately; a subclass that fades out uses it.
- **`overlaps`** — whether this effect composites with the ones below it, or replaces them and suppresses them entirely. Defaults to overlapping. This is the single most consequential bit on the type: a non-overlapping effect owns the screen.
- **`frees_on_remove`** — whether the manager destroys this effector when it is removed, or the game keeps ownership and reuses it. Defaults to the manager owning it.
- **Type** — the identity, and the one-live-per-identity rule that makes adding an effector also a removal.
