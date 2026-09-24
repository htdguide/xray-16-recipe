# src/xrGame/ParticlesPlayer.h

> Declares the mix-in that lets any skinned object play particle effects on its bones, implemented in [`ParticlesPlayer.cpp`](ParticlesPlayer.cpp.md).

**Needs** — [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md)
**Used by** — [`ExplosiveItem.cpp`](ExplosiveItem.cpp.md) · [`Mincer.cpp`](Mincer.cpp.md) · [`ParticlesPlayer.cpp`](ParticlesPlayer.cpp.md) · [`PhysicsShellHolder.cpp`](PhysicsShellHolder.cpp.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`TeleWhirlwind.cpp`](TeleWhirlwind.cpp.md) · [`entity_alive.cpp`](entity_alive.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the capability "this object can carry particle effects on named attachment points
of its skeleton". Substance is in [`ParticlesPlayer.cpp`](ParticlesPlayer.cpp.md).

The declaration's own load-bearing content is the **two-level data shape**: a list of
attachment bones, each holding a list of effects currently playing on it, each effect
tagged with who started it and how long it has left. That two-level indexing is what makes
"stop everything this weapon started" and "stop everything on this bone" both cheap, and
both of those are real operations in the game.

Exported units:

- `CParticlesPlayer` — the mix-in. Carries the bone list, the bone mask, the parent velocity
  and a flag saying whether anything is playing.
- `SBoneInfo` — one attachment point: bone index, local offset, and the effects on it, with
  find, append and two stop operations.
- `SParticlesInfo` — one playing effect: the effect object, its orientation as angles, the
  identifier of whoever started it, and its remaining lifetime with a sentinel for endless.
- `LoadParticles` — read the attachment points from the model's own configuration.
- `net_SpawnParticles` / `net_DestroyParticles` — the lifecycle bracket; the second destroys
  every live effect and is the reason a carrier's effects never outlive it.
- `UpdateParticles` — the per-frame pass: re-place, age, reap.
- `StartParticles` — four spellings: on one bone or on all of them, oriented by a direction
  or by a full transform.
- `StopParticles` — by starter identifier or by effect name, on one bone or on all.
- `AutoStopParticles` — retroactively give a running effect a lifetime.
- `get_bone_info` / `get_nearest_bone_info` / `GetNearestBone` — resolve an arbitrary bone to
  the nearest *attachment* bone by walking up the skeleton.
- `GetRandomBone` — pick an attachment point at random.
- `MakeXFORM` / `GetBonePos` — the shared world-placement helpers, usable without an
  instance, which is why they are static.
- `SetParentVel` — tell emission what velocity to inherit.
- `IsPlaying` — whether any effect is live on any bone.
- `cast_particles_player` — the capability query by which the rest of the game recognizes a
  carrier.
