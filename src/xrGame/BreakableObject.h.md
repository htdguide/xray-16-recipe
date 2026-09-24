# src/xrGame/BreakableObject.h

> Declares the two-state breakable scenery object implemented in [`BreakableObject.cpp`](BreakableObject.cpp.md).

**Needs** — [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`xrPhysics/icollisiondamagereceiver.h`](../xrPhysics/icollisiondamagereceiver.h.md)
**Used by** — [`BreakableObject.cpp`](BreakableObject.cpp.md) · [`CustomZone.cpp`](CustomZone.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the object that is one rigid piece until it breaks and a pile of bodies after.
Substance in [`BreakableObject.cpp`](BreakableObject.cpp.md).

It is both a physics-assembly holder and a **collision damage receiver**, which is what
makes it breakable by being run into rather than only by being shot.

Exported units:

- `CBreakableObject` — the object.
- `Load`, `net_Spawn`, `net_Destroy`, `shedule_Update`, `UpdateCL` — the lifecycle. The
  scheduled update exists only to run the broken pile's removal countdown.
- `Hit` — route a hit: decide whether it breaks the object, then push the pieces.
- `CollisionHit`, `PHCollisionDamageReceiver` — what the physics world calls when this
  object is struck by a moving body.
- `renderable_ShadowGenerate`, `renderable_ShadowReceive` — a breakable receives shadows and
  casts none. A pile of small pieces is not worth a shadow pass.
- `net_Export`, `net_Import`, `UsedAI_Locations` — nothing is replicated and the object holds
  no navigation position.

The private surface names the transition in full: `CreateUnbroken`, `DestroyUnbroken`,
`CreateBroken`, `ActivateBroken`, `Break`, `Split`, `ApplyExplosion`, `CheckHitBreak`,
`ProcessDamage`, `SendDestroy`. Reading them in that order *is* the object's life.

**Notes** — the four tuning values (`remove_time`, `health_threshold`, `damage_threshold`,
`immunity_factor`) are declared process-wide rather than per instance, and are overwritten
by every breakable's load. See [`BreakableObject.cpp`](BreakableObject.cpp.md).
