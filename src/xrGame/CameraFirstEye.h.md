# src/xrGame/CameraFirstEye.h

> Declares the first-person camera implemented in [`CameraFirstEye.cpp`](CameraFirstEye.cpp.md).

**Needs** — [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`CameraFirstEye.cpp`](CameraFirstEye.cpp.md) · [`Car.cpp`](Car.cpp.md) · [`CarCameras.cpp`](CarCameras.cpp.md) · [`CarInput.cpp`](CarInput.cpp.md) · [`HolderEntityObject.cpp`](HolderEntityObject.cpp.md) · [`WeaponStatMgun.cpp`](WeaponStatMgun.cpp.md) · [`spectator_camera_first_eye.cpp`](spectator_camera_first_eye.cpp.md) · [`spectator_camera_first_eye.h`](spectator_camera_first_eye.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CCameraFirstEye`. Substance is in
[`CameraFirstEye.cpp`](CameraFirstEye.cpp.md).

Exported units:

- `CCameraFirstEye` — the eye camera. Holds only a look-at target and whether it is engaged.
- `Update` — place the eye, ease any look-at, build the view basis from the aim plus noise.
- `Move` — apply rotation input, with the wrap-then-clamp discipline.
- `OnActivate` — inherit the outgoing camera's yaw when both share a reference space.
- `LookAtPoint` — engage the easing toward a world point; it disengages itself on arrival.
- `GetWorldYaw` / `GetWorldPitch` — the aim in world terms. The yaw is **negated**: the
  camera's internal yaw runs opposite to the world's, and this pair is the single place that
  conversion is stated. Every consumer of the actor's aim goes through it.
