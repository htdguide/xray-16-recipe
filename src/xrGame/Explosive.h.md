# src/xrGame/Explosive.h

> Declares the explosion behaviour any object can inherit, implemented in [`Explosive.cpp`](Explosive.cpp.md).

**Needs** — [`inventory_item.h`](inventory_item.h.md) · [`wallmark_manager.h`](wallmark_manager.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md) · [`xrEngine/Render.h`](../xrEngine/Render.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`xrPhysics/DamageSource.h`](../xrPhysics/DamageSource.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md)
**Used by** — [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md) · [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`Explosive.cpp`](Explosive.cpp.md) · [`ExplosiveItem.cpp`](ExplosiveItem.cpp.md) · [`ExplosiveItem.h`](ExplosiveItem.h.md) · [`ExplosiveRocket.cpp`](ExplosiveRocket.cpp.md) · [`ExplosiveRocket.h`](ExplosiveRocket.h.md) · [`ExplosiveScript.cpp`](ExplosiveScript.cpp.md) · [`Grenade.cpp`](Grenade.cpp.md) · [`Grenade.h`](Grenade.h.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`Helicopter2.cpp`](Helicopter2.cpp.md) · [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · _and 6 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares the mixin that makes an object able to explode. It is a **damage source**, which is
how the physics layer attributes forces it applies, and it demands one thing of whoever
inherits it: a way back to the game object (`cast_game_object`, pure). Everything else it
can do for itself.

Substance in [`Explosive.cpp`](Explosive.cpp.md).

Exported units:

- `CExplosive` — the behaviour.
- `Load` — in two forms, from the global configuration or from a caller-supplied one. The
  second exists because a vehicle's explosion parameters live in its *model's* embedded
  configuration, not in a section of the game's.
- `Explode`, `ExplodeParams`, `GenExplodeEvent`, `OnEvent` — the request/perform split: the
  authoritative host emits an event, every host performs the explosion on receipt.
- `UpdateCL` — the per-frame advance while an explosion lasts.
- `ExplosionEffect`, `TestPassEffect` — **the blast-attenuation model**: five sampled rays
  per target, weighted by presented cross-section, attenuated by distance and by every
  material passed through. The interesting page is the implementation's.
- `OnBeforeExplosion`, `OnAfterExplosion`, `HideExplosive` — the object's own disappearance,
  which a subclass overrides (a vehicle does not vanish; a grenade does).
- `GetExplPosition`, `GetExplDirection`, `GetExplVelocity`, `UpdateExplosionPos`,
  `FindNormal`, `GetRayExplosionSourcePos`, `GetExplosionBox`, `SetExplosionSize`,
  `ActivateExplosionBox` — **the subclass hooks**. These are what a moving explosive (a
  rocket), a large one (a vehicle) or an oddly shaped one overrides. In particular
  `GetRayExplosionSourcePos` decides where inside the explosion the sampling rays start,
  which is what makes a large explosion wrap around cover.
- `SetInitiator`, `SetCurrentParentID`, `CurrentParentID`, `Initiator` — who is blamed. The
  fallback when nobody is recorded is the exploding object itself.
- `IsExploding`, `IsExploded`, `IsSoundPlaying`, `Useful` — state queries. `Useful` means the
  explosion state is entirely untouched, which is the test for whether a pooled item can be
  handed out again.
- `StartLight`, `StopLight`, `UpdateExplosionParticles` — the visual half, overridable.
- `net_Destroy`, `net_Relcase` — teardown, and dropping references to objects being deleted.
- `random_point_in_object_box` — a free helper: a uniformly random point inside an object's
  bounding box, used by subclasses answering the ray-source question.

The flag set is a four-step sequence rather than four independent bits: ready to explode,
exploding, event sent, exploded.

**Notes** — the declaration includes the proximity sense and the inventory-item header, and
uses neither. Both are inherited include dependencies from an earlier arrangement in which
an explosive was always a thrown item that felt what was near it. A rebuild drops them.
