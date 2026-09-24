# src/xrEngine/ICollidable.h

> Declares "this entity has a collision shape", and the default filling of it.

**Needs** — [`xr_collide_form.h`](xr_collide_form.h.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md)
**Used by** — [`ICollidable.cpp`](ICollidable.cpp.md) · [`xr_object.h`](xr_object.h.md) · [`base_client_classes_wrappers.h`](../xrGame/base_client_classes_wrappers.h.md)
**Tier floor** — T2.

## Purpose

Collision queries — the sense of sight, weapon fire, physics contact — need to ask an arbitrary entity for its collision shape without knowing what kind of entity it is. This is the two-method interface that makes that possible, plus a base that holds the shape and registers the entity in the spatial database's collidable category.

The original marks it for merging into the general entity interface. It is separate only because it is also implemented by a few things that are not full entities.

## Exported units

- **`Collidable`** — set and get the collision shape. Two methods; nothing else.
- **`CollidableBase`** — the default filling. See [`ICollidable.cpp`](ICollidable.cpp.md) for what construction and destruction actually do, which is the only substance here.

**Notes** — The base owns its shape and destroys it. That is the convention every entity relies on: assigning a shape transfers ownership, and nothing else frees it.
