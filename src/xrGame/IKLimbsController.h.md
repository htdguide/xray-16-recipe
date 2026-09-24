# src/xrGame/IKLimbsController.h

> Declares one character's foot-placement controller, implemented in [`IKLimbsController.cpp`](IKLimbsController.cpp.md).

**Needs** — [`ik/IKLimb.h`](ik/IKLimb.h.md) · [`pose_extrapolation.h`](pose_extrapolation.h.md) · [`ik_object_shift.h`](ik_object_shift.h.md)
**Used by** — [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`IKLimbsController.cpp`](IKLimbsController.cpp.md) · [`step_manager.cpp`](step_manager.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CIKLimbsController`. Substance is in
[`IKLimbsController.cpp`](IKLimbsController.cpp.md).

Exported units:

- `CIKLimbsController` — the controller.
  - `Create`, `Destroy` — bind to and release from a character, registering and
    unregistering the pose-evaluation callback that everything else hangs off.
  - `PlayLegs` — adopt the blend currently driving the legs; footstep timing comes from
    its animation's marks.
  - `Update` — the per-frame half: advance tracks, sample the pose history, let each limb
    update its footstep timing.
  - `Shift` — the character's current vertical offset, which anything that must agree
    with where the body really is (the camera above all) has to read.
- Private: `Calculate` (the per-pose correction, called from the static callback),
  `LimbCalculate`, `LimbUpdate`, `LimbSetup`, `ShiftObject` (apply the offset to every
  bone), and the four functions that decide the offset — `ObjectShift`,
  `StaticObjectShift`, `LegLengthShiftLimit`, `PredictObjectShift`.
- `IKVisualCallback` — the static hook the skeleton calls when a pose is ready.

## Notes

The limb count is capped at four and the per-pose working array is a fixed array of that
size rather than a container, because it is built and discarded on every pose evaluation
of every visible character. That is a frame-budget decision, not a semantic limit — but
the limit is real: a creature with more than four legs is not supported.
