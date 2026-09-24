# src/xrGame/hit_immunity.h

> Declares the mix-in that gives an object a per-damage-type multiplier table.

**Needs** — [`hit_immunity_space.h`](hit_immunity_space.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`Artefact.cpp`](Artefact.cpp.md) · [`Artefact.h`](Artefact.h.md) · [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md) · [`DestroyablePhysicsObject.h`](DestroyablePhysicsObject.h.md) · [`EntityCondition.cpp`](EntityCondition.cpp.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`Wound.cpp`](Wound.cpp.md) · [`Wound.h`](Wound.h.md) · [`helicopter.h`](helicopter.h.md) · [`hit_immunity.cpp`](hit_immunity.cpp.md) · [`inventory_item.cpp`](inventory_item.cpp.md) · _and 1 more_
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`hit_immunity.cpp`](hit_immunity.cpp.md). Anything that
resists damage differently by damage type — an outfit, a helmet, a creature, a vehicle —
carries one of these.

Exported units:

- `CHitImmunity` — the table. Every slot starts at one, meaning no modification.
- `LoadImmunities` — replaces the table from a configuration section.
- `AddImmunities` — accumulates a configuration section into the table.
- `GetHitImmunity` — the multiplier for one damage type.
- `AffectHit` — a damage magnitude passed through that multiplier. The only reason the class
  exists: callers apply damage without knowing how the table was built.
