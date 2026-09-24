# src/xrEngine/Effector.h

> A camera effector: something with a lifetime that perturbs the frame's camera and then expires.

**Needs** — [`CameraDefs.h`](CameraDefs.h.md) · [`device.h`](device.h.md)
**Used by** — [`CameraManager.cpp`](CameraManager.cpp.md) · [`Effector.cpp`](Effector.cpp.md) · [`FDemoPlay.cpp`](FDemoPlay.cpp.md) · [`FDemoPlay.h`](FDemoPlay.h.md) · [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`FDemoRecord.h`](FDemoRecord.h.md) · [`AnselManager.h`](../xrGame/AnselManager.h.md) · [`CameraEffector.h`](../xrGame/CameraEffector.h.md) · [`EffectorFall.h`](../xrGame/EffectorFall.h.md) · [`SleepEffector.h`](../xrGame/SleepEffector.h.md) · [`controller_psy_hit_effector.h`](../xrGame/ai/monsters/controller/controller_psy_hit_effector.h.md) · [`pp_effector_custom.h`](../xrGame/pp_effector_custom.h.md) · [`script_effector.h`](../xrGame/script_effector.h.md)
**Tier floor** — T3: a countdown and a virtual call.

## Purpose

Camera shake, weapon recoil, a cutscene camera, a death fall, the hit reaction — all of them are the same shape: a thing that lives for a while, modifies the camera each frame while it lives, and removes itself. This is that shape. The engine defines it and implements only the countdown; every actual perturbation is the game's.

It is a base class rather than a callback because effectors carry their own accumulated state — a shake's phase, a recoil's decay — and because the stack orders them by a property (absolute vs relative positioning) that a plain callback could not report.

## State

```text
RECORD CamEffector  (extends BaseEffector)
  type          : CamEffectorType   # identity; at most one live effector per identity
  life_time     : real (seconds)    # counts down; <= 0 means expired
  affects_hud   : bool              # whether the first-person weapon model inherits the effect
```

## `apply`

**Contract** — Called once per frame with the frame's camera description, which it may modify in place. Returns whether it is still alive. The base implementation is the *entire* default behaviour: subtract this frame's elapsed time from the lifetime and report whether any is left. A subclass that overrides it must decrement the lifetime itself or run forever.

```text
FUNCTION apply(cam_info) -> bool
  life_time = life_time - frame_delta_seconds
  RETURN life_time > 0
```

**Notes** — The elapsed time used is the *scaled* frame delta, the same one the simulation advances on. An effector therefore slows down with bullet time and stops entirely while paused, which is what a camera shake attached to a game event should do. An effector that must run in real time — a menu transition — has to read the unscaled clock itself.

## `is_valid`

**Contract** — Whether lifetime remains. Separated from `apply` because the manager tests validity *before* applying, so an effector that expired between frames is never applied at all.

## `apply_expired`

**Contract** — Called instead of `apply` on an effector that has expired but has asked, via `wants_expired_processing`, to be given one last frame. Used by effectors that must restore something they changed — a field-of-view push, a held offset — rather than snapping back. The default pair is "do nothing, and no".

## `positions_absolutely`

**Contract** — Whether this effector sets the camera outright rather than perturbing it. Read once, at insertion, and it decides position in the stack: absolute effectors go to the front of the list and therefore run last, overriding every relative perturbation. Defaults to relative.

## `set_hud_affect` / `get_hud_affect`

**Contract** — Whether the first-person weapon model, which is drawn in its own pass with its own near plane, inherits this effector's perturbation. A camera shake usually should — the weapon shakes with the view — while a cutscene camera should not, because the weapon is not visible at all.
