# src/xrEngine/CameraBase.h

> The abstract camera: an owner, an orientation expressed as yaw/pitch/roll with optional limits, and a basis the game may set directly instead.

**Needs** — [`CameraDefs.h`](CameraDefs.h.md) · [`device.h`](device.h.md) · [`xr_object.h`](xr_object.h.md)
**Used by** — [`CameraBase.cpp`](CameraBase.cpp.md) · [`CameraManager.cpp`](CameraManager.cpp.md) · [`ActorCameras.cpp`](../xrGame/ActorCameras.cpp.md) · [`ActorMountedWeapon.cpp`](../xrGame/ActorMountedWeapon.cpp.md) · [`Actor_Feel.cpp`](../xrGame/Actor_Feel.cpp.md) · [`Actor_Movement.cpp`](../xrGame/Actor_Movement.cpp.md) · [`AnselManager.h`](../xrGame/AnselManager.h.md) · [`CameraFirstEye.cpp`](../xrGame/CameraFirstEye.cpp.md) · [`CameraFirstEye.h`](../xrGame/CameraFirstEye.h.md) · [`CameraLook.h`](../xrGame/CameraLook.h.md) · [`HudItem.cpp`](../xrGame/HudItem.cpp.md) · [`Missile.cpp`](../xrGame/Missile.cpp.md) · [`Torch.cpp`](../xrGame/Torch.cpp.md) · [`actor_memory.cpp`](../xrGame/actor_memory.cpp.md) · _and 4 more_
**Tier floor** — T3: pure state and arithmetic. It is an interface the game fills in, with no device contact.

## Purpose

Every camera in the game — first person, third person, free-flight debug, cinematic — is one of these. The engine defines the shape so that the camera manager can drive any of them uniformly and the level can hand one to the device, while every actual behaviour (how a camera follows a body, how it collides with walls) lives in the game module.

The contract is deliberately double-ended. A camera carries *both* an Euler orientation and an explicit position/direction/up basis, and different subclasses treat one or the other as authoritative. This is not redundancy the rebuilder may collapse: a follow camera integrates yaw and pitch from mouse deltas and derives the basis, while a cinematic camera is handed a basis from an animation track and has no meaningful Euler angles. The manager reads only the basis, and it is each camera's job to make the basis current in `update`.

## State

```text
RECORD Camera
  owner            : game_object         # invariant: never absent; a camera without an owner is a bug
  flags            : set of {relative_link, position_rigid, direction_rigid}
  tag              : int                 # game-defined discriminator; see Notes

  clamp_yaw        : bool
  clamp_pitch      : bool
  clamp_roll       : bool
  yaw, pitch, roll : real (radians)
  lim_yaw          : pair<real, real>    # low, high
  lim_pitch        : pair<real, real>
  lim_roll         : pair<real, real>
  rot_speed        : vector3             # radians per unit of input, one per axis

  position         : vector3
  direction        : vector3             # default (0,0,1)
  up               : vector3             # default (0,1,0)
  fov              : real (degrees)      # default 90
  aspect           : real                # default 1

  saved_*          : a full copy of the six orientation fields above
```

The three flags are read by the camera manager, not by the camera:

- **relative_link** — the camera's transform is expressed relative to its owner rather than in world space.
- **position_rigid** — the manager must snap to this camera's position instead of smoothing toward it. Set by cameras whose motion is already authored (cutscenes, vehicle mounts); clear on cameras the player drives, where the smoothing is the feel.
- **direction_rigid** — the same for orientation.

`right` is never stored; it is the cross product of up and direction, recomputed on demand. Storing it would create a fourth field to keep consistent for no gain.

## `load`

**Contract** — Implemented in [`CameraBase.cpp`](CameraBase.cpp.md). Reads rotation speed and the yaw/pitch limits from a named configuration section and derives the clamp flags and a starting orientation from them.

## `activate` / `deactivate`

**Contract** — Called when this camera becomes or stops being the active one. Activation receives the outgoing camera so a subclass can inherit its orientation and avoid a visible jump on the switch; the default does nothing, which is correct for a camera that computes its orientation from scratch every frame.

## `move`

**Contract** — Receives one movement or rotation command with a magnitude and a multiplier, from the input layer's action mapping. The command set is the game's, not the engine's. The default ignores everything, which makes a non-interactive camera a subclass that implements nothing.

## `update`

**Contract** — Advances the camera for this frame around an anchor point, and receives a noise/shake angle triple to fold in. On return the position/direction/up basis must be current, because that is all the manager reads. The default does nothing, so a camera that is positioned entirely by external code needs no implementation.

## `get` / `set`

**Contract** — Read or overwrite the basis, or overwrite the Euler triple. Both setters exist because both representations are authoritative for some subclass; neither derives the other at this level.

## `check_limit_yaw` / `check_limit_pitch` / `check_limit_roll`

**Contract** — Report how close the corresponding angle is to its authored limit, as a signed normalized value. Implemented in [`CameraBase.cpp`](CameraBase.cpp.md). Used by the game to push back on the player's aim near a limit rather than letting it stop dead.

## `save_orientation` / `restore_orientation`

**Contract** — A one-deep, non-nesting stash of the entire orientation (Euler triple plus basis). The game uses it to run a temporary camera override — an animation, a scripted look-at — and come back to exactly where the player was pointing. One slot only: a second save overwrites the first, and the game is expected not to nest.

## `world_yaw` / `world_pitch`

**Contract** — The orientation in world terms for a camera whose stored angles are relative to a moving owner. The default reports zero, meaning "my angles are already world angles".

## `viewport_half_extents`

**Contract** — Given a near-plane distance and anything that reports a field of view, produce the half-height and half-width of that plane. Used wherever the engine needs the near rectangle itself — frustum construction, projected decals, the first-person weapon's separate near plane.

**Notes** — Height comes from the field of view (the engine's fov is *vertical*) and width is derived by dividing by the aspect ratio. The aspect used is deliberately the **device's** current aspect, not the camera's own — the camera's stored aspect lags behind by a frame because the manager smooths it, and a lagging aspect would make the computed near rectangle disagree with the projection matrix actually in use.

**Notes** — `tag` and the flag set overlap in intent: both are "which kind of camera is this" markers consulted by game code. The original marks this as a known wart. A rebuild should have one discriminator.
