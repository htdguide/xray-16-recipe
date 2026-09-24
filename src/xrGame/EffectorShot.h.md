# src/xrGame/EffectorShot.h

> Declares the recoil model and its camera-chain wrapper, implemented in [`EffectorShot.cpp`](EffectorShot.cpp.md).

**Needs** — [`CameraRecoil.h`](CameraRecoil.h.md) · [`CameraEffector.h`](CameraEffector.h.md) · [`Actor.h`](Actor.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — [`ActorCameras.cpp`](ActorCameras.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`EffectorShot.cpp`](EffectorShot.cpp.md) · [`EffectorShotX.h`](EffectorShotX.h.md) · [`WeaponDispersion.cpp`](WeaponDispersion.cpp.md) · [`WeaponFire.cpp`](WeaponFire.cpp.md) · [`WeaponStatMgunFire.cpp`](WeaponStatMgunFire.cpp.md) · [`ai_stalker_impl.h`](ai/stalker/ai_stalker_impl.h.md) · [`object_handler.cpp`](object_handler.cpp.md) · [`stalker_animation_callbacks.cpp`](stalker_animation_callbacks.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the two accumulating recoil angles, their per-frame deltas, the burst counter and
the state flags, plus the effector that carries them into the camera chain. Substance in
[`EffectorShot.cpp`](EffectorShot.cpp.md).

Exported units:

- `CWeaponShotEffector` — the recoil model itself, usable without a camera.
- `Initialize`, `Reset` — install a weapon's tuning; clear the accumulators.
- `Shot` — one round fired, deriving the kick from the burst count and the silencer.
- `Shot2` — apply one kick directly, for shooters that do not go through a weapon.
- `Update` — the once-per-frame tick: relax, then latch the deltas.
- `StopShoting`, `IsActive` — trigger release, and whether recoil is still contributing.
- `GetDeltaAngle` — the accumulated offset, for displacing the camera.
- `GetLastDelta` — this frame's change, for moving the aim.
- `ChangeHP` — apply this frame's change to a pitch/yaw pair in place.
- `SetRndSeed` — seed the local random source; latches on first call and ignores its
  argument (see the implementation's notes).
- `Relax` — protected: one frame of return, with the horizontal rate matched to the
  vertical.
- `CCameraShotEffector` — the model as a camera-chain effector, tagged with the weapon it
  belongs to and able to identify itself to the character by a type query.
