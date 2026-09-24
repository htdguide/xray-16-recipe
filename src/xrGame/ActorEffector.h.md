# src/xrGame/ActorEffector.h

> Declares the actor's effector vocabulary: the strength-source interface, the animation-driven camera effectors, and the actor's two-camera effector manager.

**Needs** — [`CameraEffector.h`](CameraEffector.h.md) · [`ActorEffector.cpp`](ActorEffector.cpp.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`ActorEffector.cpp`](ActorEffector.cpp.md) · [`ActorEffector_script.cpp`](ActorEffector_script.cpp.md) · [`ActorMountedWeapon.cpp`](ActorMountedWeapon.cpp.md) · [`ActorVehicle.cpp`](ActorVehicle.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`Explosive.cpp`](Explosive.cpp.md) · [`HolderEntityObject.cpp`](HolderEntityObject.cpp.md) · [`Missile.cpp`](Missile.cpp.md) · [`PostprocessAnimator.cpp`](PostprocessAnimator.cpp.md) · _and 15 more_
**Tier floor** — T3: a declaration

## Purpose

Declares the classes implemented in [`ActorEffector.cpp`](ActorEffector.cpp.md) and
[`ActorEffector_script.cpp`](ActorEffector_script.cpp.md). Its own load-bearing content is
the **taxonomy**: what kinds of thing can drive a camera effect, which is the map a reader
needs before the implementation reads as anything but four near-identical installers.

Exported units:

- `CActorCameraManager` — the actor's effector stack. Adds the second camera pose the
  first-person weapon model is drawn from, and a query returning it as a transform.
- `AddEffector` in four forms and `RemoveEffector` — install a configuration-described
  effect on the actor with no strength source, a constant, a per-frame function, or a
  controller object; remove by effect type.
- `CEffectorController` — the interface a caller implements to own an effect's lifetime and
  strength. It holds back-pointers to the camera and screen effectors it drives and demands
  that both be cleared before it dies.
- `CAnimatorCamEffector` — a recorded camera animation applied relative to the current view,
  or absolutely when the positioning flag is set. Optionally overrides the field of view.
- `CAnimatorCamEffectorScriptCB` — the same, plus a script function called once when the
  animation ends. Its behaviour when invalid is to keep positioning the camera if it is in
  absolute mode, which is what lets a cutscene hold its final framing.
- `CAnimatorCamLerpEffector` and its constant-strength variant — the animation blended
  toward the unmodified view by a strength in the unit range.
- `CCameraEffectorControlled` — the blended form driven by a controller.
- `SndShockEffector` — the deafening blast: mutes and restores global sound volume while
  running the paired hit effector.
- `CControllerPsyHitCamEffector` — the psychic attack's camera flight toward the attacker,
  with a jitter of half a degree per axis at a fixed angular speed.

## Notes

**Effect type doubles as identity.** An effector is installed under a type tag and removed
by that tag, so at most one effect of each type exists at a time and a second install
replaces the first. The tag set is shared with the engine's effector stack, so the game
cannot invent tags freely.

**The strength function is a bound callable.** In a rebuild it is simply "a function
returning a number in 0..1, evaluated each frame"; the binding mechanism is incidental.
