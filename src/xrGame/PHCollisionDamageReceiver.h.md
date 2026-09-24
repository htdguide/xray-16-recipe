# src/xrGame/PHCollisionDamageReceiver.h

> Declares the collision-to-damage mixin implemented in [`PHCollisionDamageReceiver.cpp`](PHCollisionDamageReceiver.cpp.md).

**Needs** — [`xrPhysics/icollisiondamagereceiver.h`](../xrPhysics/icollisiondamagereceiver.h.md) · [`PHCollisionDamageReceiver.cpp`](PHCollisionDamageReceiver.cpp.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md) · [`DestroyablePhysicsObject.h`](DestroyablePhysicsObject.h.md) · [`PHCollisionDamageReceiver.cpp`](PHCollisionDamageReceiver.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CPHCollisionDamageReceiver`, the mixin an entity inherits to take damage from
physical collisions on specific bones. Substance is in
[`PHCollisionDamageReceiver.cpp`](PHCollisionDamageReceiver.cpp.md).

The one thing the declaration itself decides is the shape of the contract with the inheriting
class: the mixin demands a single accessor returning the physics-bearing object, and gets
everything else — the model, the skeleton, the physics body, the entity identifier — through
it. That is the whole coupling, and a rebuild should keep it that narrow.

Exported units:

- `CPHCollisionDamageReceiver` — the mixin, implementing the physics layer's damage-receiver
  interface and holding the per-bone factor table.
- `PPhysicsShellHolder` — the one thing an implementor must supply.
- `Init` — read the model's embedded damage table and subscribe the listed bones.
- `Clear` — empty the table.
- `CollisionHit` — convert one contact into a hit event. Reached through the physics layer's
  callback, not called directly.
- `BoneInsert` / `FindBone` (private) — table maintenance and linear lookup.
