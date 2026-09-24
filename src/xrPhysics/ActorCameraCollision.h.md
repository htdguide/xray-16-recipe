# src/xrPhysics/ActorCameraCollision.h

> The two questions the camera asks physics — "would the near plane be inside
> something here?" and "push me out of it".

**Needs** — [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md)
**Used by** — [`ActorCameras.cpp`](../xrGame/ActorCameras.cpp.md) · [`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md)
**Tier floor** — T2: a three-call boundary; nothing here touches bytes.

## Purpose

Declares the surface implemented in
[`ActorCameraCollision.cpp`](ActorCameraCollision.cpp.md). The game layer owns the camera
and physics owns the world, so the camera cannot simply ask "am I clear" without a
collision query; these three calls are the whole of that conversation.

The module-level handle on the one persistent camera-collision shell is also published
here, so the game can destroy it when the level unloads. Everything else on this header
exists only under a debug build: a draw toggle and two tunables that were left live so they
could be dialled in while the game ran.

## exported units

- **`test_camera_box`** — does a box of the given size at the given transform touch
  anything the camera should not see through? Reports only; moves nothing.
- **`test_camera_collide`** — the same question phrased in camera terms: build the box
  from the camera's near plane, scale it, offset it along the view direction, ask.
- **`collide_camera`** — the acting form: build the near-plane box, and if it is embedded,
  move the camera out and write the corrected position back into the camera.
- **`actor_camera_shell`** — the shared collision body the three calls reuse.
