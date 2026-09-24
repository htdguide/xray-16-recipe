# src/xrGame/Bolt.h

> Declares the bolt implemented in [`Bolt.cpp`](Bolt.cpp.md).

**Needs** — [`Missile.h`](Missile.h.md) · [`xrPhysics/DamageSource.h`](../xrPhysics/DamageSource.h.md)
**Used by** — [`Bolt.cpp`](Bolt.cpp.md) · [`InventoryOwner.cpp`](InventoryOwner.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CBolt`, the infinite throwable used to probe for anomalies. Substance is in
[`Bolt.cpp`](Bolt.cpp.md).

Exported units:

- `CBolt` — a thrown item that is also a damage source. Holds the thrower's entity
  identifier.
- `Throw` — throw it and immediately spawn its replacement.
- `Useful` — always false, so it is never picked up from the ground.
- `activate_physic_shell` — give the flying bolt near-zero air resistance.
- `SetInitiator` / `Initiator` — record and read who threw it, for anomaly attribution.
- `OnH_A_Chield` — take the thrower from the grandparent when the bolt is acquired.
- `UsedAI_Locations` — declares false: a bolt is never placed on the navigation mesh.
- `cast_IDamageSource` — the capability query that says this object can be blamed for damage.
