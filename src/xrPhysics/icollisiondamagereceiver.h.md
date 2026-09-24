# src/xrPhysics/icollisiondamagereceiver.h

> The port through which a collision becomes damage: whatever can be hurt by being hit
> implements this, and physics calls it.

**Needs** — [`xrPhysics.h`](xrPhysics.h.md) · [`collisiondamagereceiver.cpp`](collisiondamagereceiver.cpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`BreakableObject.cpp`](../xrGame/BreakableObject.cpp.md) · [`BreakableObject.h`](../xrGame/BreakableObject.h.md) · [`PHCollisionDamageReceiver.cpp`](../xrGame/PHCollisionDamageReceiver.cpp.md) · [`PHCollisionDamageReceiver.h`](../xrGame/PHCollisionDamageReceiver.h.md) · [`collisiondamagereceiver.cpp`](collisiondamagereceiver.cpp.md)
**Tier floor** — T2: one method and two callback declarations.

## Purpose

Physics knows how hard something was hit; it does not know what "hurt" means. This interface
is the entire boundary: one method, implemented by anything that can take collision damage —
a creature, the player, a breakable prop — and called from inside the contact callback.

Keeping it to one method matters. The receiver is invoked *during* collision detection, so
the implementation must not create, destroy or move anything; it may only record. Every
shipped implementation queues the damage and applies it after the step.

## `ICollisionDamageReceiver`

**Contract** — an implementor must accept:

```text
FUNCTION collision_hit(source_id, bone_id, power, direction, position)
  # source_id : the entity that did the hitting, or the "no entity" marker when
  #             the damager is static geometry or anonymous
  # bone_id   : which of the receiver's own bones was struck, or the "no bone"
  #             marker for a whole-object hit
  # direction : the contact normal
  # position  : the contact point RELATIVE TO the struck shape's origin —
  #             not a world position, and not a bone-space position either
```

**Invariants** — the implementation must be re-entrant with respect to the physics step: it
may read the world but must not mutate it. It may be called many times in one step, once per
contact.

**Notes** — the position argument is the honest weak point, and the original says so: it is
the contact point minus the struck *shape's* origin, which is neither world space nor the
bone space the caller usually wants. Consumers that need bone space must compose the shape's
placement themselves. A rebuild should pass world position and let the receiver transform it.

## `DamageReceiverCollisionCallback` / `BreakableObjectCollisionCallback`

**Contract** — declared here, implemented in
[`collisiondamagereceiver.cpp`](collisiondamagereceiver.cpp.md). Both are object-contact
callbacks in the sense of [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md), installed
on the shapes of whatever wants to receive collision damage.
