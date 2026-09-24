# src/xrEngine/CameraDefs.h

> The camera's value types: the per-frame camera description that effectors mutate, and the identity of the two effector stacks.

**Needs** — [`device.h`](device.h.md) · [`xrMiscMath`](../utils/xrMiscMath/README.md)
**Used by** — [`CameraBase.h`](CameraBase.h.md) · [`CameraManager.h`](CameraManager.h.md) · [`Effector.h`](Effector.h.md) · [`EffectorPP.h`](EffectorPP.h.md) · [`AnselManager.cpp`](../xrGame/AnselManager.cpp.md)
**Tier floor** — T3: a record of floats and two enumerations; nothing here touches a device.

## Purpose

Camera state travels through the frame as one flat record rather than as a chain of getters, because the camera-effector stack is a *pipeline*: each effector reads the record, perturbs it and hands it on. Making the record a single value makes "apply a list of effects in order" trivially expressible and makes the stack's ordering rule visible.

## State

```text
RECORD CameraInfo
  p                : vector3          # eye position in world space
  d                : vector3          # forward, unit;  default (0,0,1)
  n                : vector3          # up, unit;       default (0,1,0)
  r                : vector3          # right; derived, never authored
  fov              : real (degrees)   # default 90
  near             : real             # fixed at the engine's near plane, 0.2
  far              : real             # default 100, overwritten from the weather's far plane
  aspect           : real
  offset_x         : real             # projection shear, for external screenshot tooling
  offset_y         : real
  dont_apply       : bool             # an effector may claim the frame and suppress the device write
  affected_on_hud  : bool             # whether the first-person weapon model inherits the effect

  # invariant after any update: d, n, r are mutually orthogonal and unit length.
  # Effectors are allowed to break this; the manager re-orthogonalizes after the stack runs.
```

`near` is a constant of the engine, not of the camera — the whole renderer's depth precision is tuned around it, so an effector that wants a different near plane does not get one. A second, much closer near plane exists for the first-person weapon model, which is drawn in its own pass to keep it out of the world's depth range.

## `BaseEffector`

**Contract** — The common root of both effector families. It carries exactly one thing: a callback invoked when the effector is removed from its stack, before the effector is destroyed. Game code registers a removal callback to learn that a cinematic camera move or a screen effect has finished, since neither family reports completion any other way.

**Notes** — The callback is the only reason a common base exists; everything else about a camera effector and a post-process effector is different. In a rebuild, "notify on removal" is a property of the stack entry rather than a base class.

## `CamEffectorType`

**Contract** — Identity for a camera effector. Two identities are fixed by the engine — the demo-playback camera and the external screenshot-tool camera — and everything above a fixed threshold is handed out dynamically on request. The identity is also the removal key: adding an effector with an identity already present removes the old one first, so an identity is at most one live effector.

## `EffectorPPType`

**Contract** — Identity for a post-process effector, with the same one-live-per-identity rule and the same dynamic allocation above the threshold. The engine reserves none of these for itself; the game owns the whole space.

**Notes** — The dynamic-identity scheme exists because the set of effects is a game-layer decision and the engine must not hold a table of them. The threshold is large enough that no fixed identity can ever collide with an allocated one; the exact value is arbitrary.
