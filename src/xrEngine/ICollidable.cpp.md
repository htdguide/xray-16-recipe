# src/xrEngine/ICollidable.cpp

> Registers a collidable entity into the spatial database's collidable category, and owns its shape.

**Needs** — [`ICollidable.h`](ICollidable.h.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md) · [`xr_collide_form.h`](xr_collide_form.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

Two lines of real decision, both about lifetime and registration.

## `CollidableBase` construction

**Contract** — Starts with no shape, and — if this object is also a spatial-database participant — adds the collidable bit to its spatial category mask. That bit is what makes the object answer collision queries at all; a spatial object without it is invisible to every ray cast.

**Notes** — The test for spatial participation is a dynamic downcast performed *during construction*, at a point where the derived object is not yet complete. That is what forces this type to keep its dispatch table (the original notes it as something to fix). The decision underneath is simple — "if I am also in the spatial database, mark me collidable" — and a rebuild that composes rather than inherits sets the bit explicitly at registration and deletes the problem.

## `CollidableBase` destruction

**Contract** — Destroys the collision shape. Ownership of a shape passes to the collidable when it is set, and this is the only place a shape is released.
