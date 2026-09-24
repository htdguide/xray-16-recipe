# src/xrPhysics/ExtendedGeom.cpp

> The three shape-payload helpers that cannot be resolved at the point of use.

**Needs** — [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`dcylinder/dCylinder.h`](dcylinder/dCylinder.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: shape-class identity and pointer unwrapping across the module boundary.

## Purpose

Almost all of [`ExtendedGeom.h`](ExtendedGeom.h.md) is inline. Three things cannot be: the
two that must be callable from outside this module, and one that needs to know about the
engine's own cylinder shape class. The split is a build constraint, not a design; a rebuild
merges the file into its header.

## `get_user_data`

**Contract** — given a contact and a flag saying which of the contact's two shapes is
"mine", return the two payload records with mine first. Either may be absent when the
corresponding shape is the level's static mesh.

**Notes** — every contact callback in the chapter starts with this call, and the flag it
takes is the reason: the solver reports a contact with its two shapes in an order the
callback did not choose, and a callback that gets the orientation wrong applies its
friction, its damage and its normal direction backwards. Normalizing the order once, here,
is what makes the callbacks readable.

## `PHRetrieveGeomUserData`

**Contract** — the exported form of shape-transform unwrapping plus payload lookup, for
callers outside this module.

## `IsCyliderContact`

**Contract** — reports whether the *moving* side of a contact is a cylinder: it inspects
whichever of the contact's two shapes has a body behind it, unwraps any transform, and
compares the shape class against the engine's own cylinder class.

**Notes** — debug-only, and the reason it exists is worth recording: the cylinder shape is
the engine's own addition to the dynamics library
([`dcylinder/dCylinder.cpp`](dcylinder/dCylinder.cpp.md)), so cylinder contacts come from
code the library's authors never wrote and are the first suspect when a contact looks wrong.
The predicate is there to filter a debug view down to them.
