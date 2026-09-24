# src/xrEngine/CameraManager.h

> Declares the camera manager and the two module-wide post-process reference values.

**Needs** — [`CameraDefs.h`](CameraDefs.h.md) · [`xrCore/PostProcess/PPInfo.hpp`](../xrCore/PostProcess/PPInfo.hpp.md)
**Used by** — [`CameraManager.cpp`](CameraManager.cpp.md) · [`EffectorPP.cpp`](EffectorPP.cpp.md) · [`FDemoPlay.cpp`](FDemoPlay.cpp.md) · [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`IGame_Level.cpp`](IGame_Level.cpp.md) · [`ActorEffector.cpp`](../xrGame/ActorEffector.cpp.md) · [`AnselManager.cpp`](../xrGame/AnselManager.cpp.md) · [`CameraEffector.h`](../xrGame/CameraEffector.h.md) · [`CameraLook.cpp`](../xrGame/CameraLook.cpp.md) · [`CarCameras.cpp`](../xrGame/CarCameras.cpp.md) · [`EffectorBobbing.cpp`](../xrGame/EffectorBobbing.cpp.md) · [`EffectorBobbing.h`](../xrGame/EffectorBobbing.h.md) · [`EffectorShot.h`](../xrGame/EffectorShot.h.md) · [`EffectorZoomInertion.cpp`](../xrGame/EffectorZoomInertion.cpp.md) · _and 9 more_
**Tier floor** — T2: a declaration surface over the state described in the implementation.

## Purpose

Declares the surface implemented in [`CameraManager.cpp`](CameraManager.cpp.md), plus the shared values that file defines.

## Exported units

- **`CameraManager`** — owns the frame's camera description and the two effector stacks. Constructed with a flag saying whether it commits to the device inside `update` or leaves the commit to its caller; the game's own manager suppresses the automatic commit so it can run extra passes between the stacks and the device.
- **Effector stack operations** — add, get, remove and identity allocation for both stacks; `count` reports live plus staged camera effectors.
- **Accessors** — position, direction, up, right, field of view, aspect, and the camera basis as a matrix.
- **`update` / `update_from_camera` / `apply_to_device` / `reset_post_process` / `dump`** — see the implementation twin.
- **`identity` and `zero` post-process sets** — the neutral and all-zero reference values the compositing rule is expressed against.
- **Camera inertia and slide inertia** — module-wide smoothing constants, both console-settable.
- **The custom effector identity threshold** — the value above which identities are allocated dynamically.

**Notes** — Three of the stack steps are overridable, and the game overrides the per-effector step to add its own validity rules. A rebuild wanting to avoid inheritance can make the step a supplied policy instead; nothing else about the manager is extended.
