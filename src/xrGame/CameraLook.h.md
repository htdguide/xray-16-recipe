# src/xrGame/CameraLook.h

> Declares the three third-person cameras implemented in [`CameraLook.cpp`](CameraLook.cpp.md).

**Needs** — [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`CameraLook.cpp`](CameraLook.cpp.md) · [`Car.cpp`](Car.cpp.md) · [`CarCameras.cpp`](CarCameras.cpp.md) · [`CarInput.cpp`](CarInput.cpp.md) · [`Spectator.cpp`](Spectator.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the orbit camera and its two variants. Substance is in
[`CameraLook.cpp`](CameraLook.cpp.md).

Exported units:

- `CCameraLook` — the orbit camera. Holds the authored distance limits, the requested
  distance and the smoothed actual distance after collision.
  - `Load` — read the limits; start at their midpoint.
  - `Update` / `UpdateDistance` — build the basis, then shorten the orbit distance against
    whatever the world puts behind the player.
  - `Move` — zoom and rotate, clamped.
  - `OnActivate` — inherit the outgoing camera's yaw and position.
  - `GetWorldYaw` / `GetWorldPitch` — the aim in world terms, with the yaw negated; the same
    convention conversion as the first-person camera's.
- `CCameraLook2` — the orbit with a fixed shoulder offset and a hold-to-lock auto-aim onto
  the nearest visible living creature. Adds two inertia pairs (minimum and maximum easing
  speed, per axis) and the shared offset vector.
- `CCameraFixedLook` — a camera that takes no input and eases to a fixed framing a quarter
  turn from wherever it was activated, holding it thereafter. Stores its current and final
  orientations as rotations rather than angles, because it interpolates between them.

## Notes

**The offset vector on the second variant is shared by every instance**, not per camera. See
[`CameraLook.cpp`](CameraLook.cpp.md); a rebuild should make it a field.
