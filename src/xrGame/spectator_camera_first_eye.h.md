# src/xrGame/spectator_camera_first_eye.h

> Declares the spectator's first-person camera, which takes its frame time from its owner rather than from the global clock.

**Needs** — [`CameraFirstEye.h`](CameraFirstEye.h.md) · [`xrCore/FTimer.h`](../xrCore/FTimer.h.md)
**Used by** — [`Spectator.cpp`](Spectator.cpp.md) · [`spectator_camera_first_eye.cpp`](spectator_camera_first_eye.cpp.md)
**Tier floor** — T3: a declaration over one borrowed value

## Purpose

Declares the surface implemented in [`spectator_camera_first_eye.cpp`](spectator_camera_first_eye.cpp.md).

## Exported units

- `Move` — apply one look command, scaling the derived step by the borrowed frame time.

## Notes

The borrowed frame time is held by reference and copying is explicitly forbidden, which
together state the lifetime rule: the camera may not outlive the spectator controller that
owns the value it reads. A rebuild without reference semantics needs some other way to say
the same thing — a handle back to the owner, or a supplier the camera calls.
