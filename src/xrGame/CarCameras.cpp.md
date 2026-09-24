# src/xrGame/CarCameras.cpp

> The three vehicle cameras and the one rule that distinguishes them: in first person the rider's head follows the camera, everywhere else it does not.

**Needs** — [`Car.h`](Car.h.md) · [`Actor.h`](Actor.h.md) · [`CameraLook.h`](CameraLook.h.md) · [`CameraFirstEye.h`](CameraFirstEye.h.md) · [`Level.h`](Level.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: transform composition and one branch

## Purpose

A vehicle offers three views — first person, chase and free — and they are three separately
constructed cameras rather than one camera with three modes, because they differ in how
they are *attached*: the first-person camera is rigidly linked to the car's transform, the
chase camera follows it with lag, and the free camera is not linked at all. Each loads its
own limits and speeds from its own configuration section.

## State

`Stateless.` The three cameras and the active selection live on the vehicle.

## `cam_Update`

**Contract** — place the active camera for this frame and hand the result to the level's
camera manager. Must not be called from inside a physics step.

```text
FUNCTION cam_update(dt, fov)
  point = the vehicle's transform applied to the authored eye position
  IF the active camera is first-person AND an actor is riding THEN
    write the camera's yaw and pitch back onto the rider, negated
  set the camera's field of view
  advance the camera from point with no additional rotation
  hand the camera to the level's camera manager
```

**Invariants** — the negated write-back is the whole point of the function. In first
person the *rider's head follows the camera*, so that leaning out and the driver's visible
pose agree with where the player is looking. In the other two views the rider is drawn
normally and the camera moves independently. Losing this makes a first-person driver look
permanently forward while the view swings.

## `HUDView`

**Contract** — the heads-up display is drawn only in the first-person view.

## `OnCameraChange`

**Contract** — select one of the three cameras. Also decides whether the rider's model is
drawn.

```text
FUNCTION on_camera_change(type)
  IF there is a rider THEN
    IF switching TO first-person    THEN hide the rider
    ELSE IF switching FROM first-person THEN show the rider
  IF this is actually a change THEN
    activate the chosen camera
    IF it is the free camera THEN seed its yaw from the vehicle's current heading
```

**Notes** — the rider is hidden in first person because the camera sits inside their head
and would otherwise be looking at the inside of their own model. The free camera is seeded
from the vehicle's heading so that switching to it does not snap the view to wherever it
was left last time.
