# src/xrGame/EffectorBobbing.h

> Declares the walk-cycle camera effector implemented in [`EffectorBobbing.cpp`](EffectorBobbing.cpp.md).

**Needs** — [`CameraEffector.h`](CameraEffector.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — [`EffectorBobbing.cpp`](EffectorBobbing.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the head-bob effector: the six configured gait parameters, the continuous phase
clock, the fade factor and the pushed-in movement state. Substance in
[`EffectorBobbing.cpp`](EffectorBobbing.cpp.md).

Exported units:

- `CEffectorBobbing` — a permanent camera-chain effector; it is silenced by fading, not
  by expiring.
- `SetState` — the owner pushes in movement flags, limping and aiming each frame.
- `ProcessCam` — the per-frame perturbation of the camera position and basis.

**Notes** — two declared fields (a standing amplitude and a standing speed) are written at
construction and never read. They are leftovers from a version whose parameters were not
in configuration.
