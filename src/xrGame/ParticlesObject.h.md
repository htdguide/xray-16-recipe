# src/xrGame/ParticlesObject.h

> Declares the world-entity wrapper around one playing particle effect, implemented in [`ParticlesObject.cpp`](ParticlesObject.cpp.md).

**Needs** — [`xrEngine/PS_instance.h`](../xrEngine/PS_instance.h.md)
**Used by** — [`BastArtifact.cpp`](BastArtifact.cpp.md) · [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md) · [`Bolt.cpp`](Bolt.cpp.md) · [`Car.cpp`](Car.cpp.md) · [`CarExhaust.cpp`](CarExhaust.cpp.md) · [`CustomRocket.cpp`](CustomRocket.cpp.md) · [`CustomZone.cpp`](CustomZone.cpp.md) · [`Explosive.cpp`](Explosive.cpp.md) · [`Explosive.h`](Explosive.h.md) · [`GamePersistent.cpp`](GamePersistent.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md) · [`Level_load.cpp`](Level_load.cpp.md) · _and 14 more_
**Tier floor** — T3: a declaration

## Purpose

Declares the particle-effect object and, more usefully, fixes the **only two sanctioned ways
to obtain and release one**: a static create call and a static destroy call that clears the
caller's handle. Nothing in the game constructs or deletes one directly, because retirement
is deferred through the engine's particle registry. Substance is in
[`ParticlesObject.cpp`](ParticlesObject.cpp.md).

It also publishes a shared zero-velocity constant, used everywhere an effect is parented to
something that is not moving — worth noting only because it is the default for the parenting
call and a rebuild will otherwise invent a per-call-site zero.

Exported units:

- `CParticlesObject` — the object. A renderer-owned emitter plus a spatial-index entry, a
  scheduler entry, a lifetime and an auto-remove rule.
- `Create` / `Destroy` — the ownership pair. Destroy retires through the registry and nulls
  the caller's handle.
- `Play` / `play_at_pos` / `Stop` — start where you are, start at a point, stop with or
  without letting live particles finish.
- `SetXFORM` / `UpdateParent` / `XFORM` / `Position` — placement. The first two differ in
  whether existing particles move with the effect and whether emission inherits velocity.
- `shedule_Needed` / `shedule_Scale` / `shedule_Update` — scheduler participation. Always
  needed; priority falls off with camera distance.
- `renderable_Render` — submit to the render queue, advancing the emitter first.
- `PerformAllTheWork` / `PerformAllTheWork_mt` — the immediate advance and its worker-thread
  twin. The threaded one belongs to a disabled parallel path.
- `Locked` — whether a threaded advance is outstanding. Serves only that disabled path.
- `IsPlaying` / `IsLooped` / `IsAutoRemove` / `SetAutoRemove` / `Name` — queries. `IsPlaying`
  stays true through a deferred stop until the last particle dies, which is a different
  question from whether the object is alive.
