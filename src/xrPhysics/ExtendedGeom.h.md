# src/xrPhysics/ExtendedGeom.h

> Declares the payload the engine staples onto every collision shape — the
> material, the owning object, the callbacks, and the per-shape triangle cache.

**Needs** — [`PHObject.h`](PHObject.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`xrCDB/xrCDB.h`](../xrCDB/xrCDB.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`BastArtifact.cpp`](../xrGame/BastArtifact.cpp.md) · [`CarWheels.cpp`](../xrGame/CarWheels.cpp.md) · [`CustomRocket.cpp`](../xrGame/CustomRocket.cpp.md) · [`Helicopter2.cpp`](../xrGame/Helicopter2.cpp.md) · [`Missile.cpp`](../xrGame/Missile.cpp.md) · [`PHDebug.cpp`](../xrGame/PHDebug.cpp.md) · [`character_shell_control.cpp`](../xrGame/character_shell_control.cpp.md) · [`imotion_position.cpp`](../xrGame/imotion_position.cpp.md) · [`interactive_animation.cpp`](../xrGame/interactive_animation.cpp.md) · [`physics_game.cpp`](../xrGame/physics_game.cpp.md) · [`CalculateTriangle.h`](CalculateTriangle.h.md) · [`ExtendedGeom.cpp`](ExtendedGeom.cpp.md) · [`Geometry.cpp`](Geometry.cpp.md) · [`Geometry.h`](Geometry.h.md) · _and 27 more_
**Tier floor** — T1: the record is reached through the pointer slot the dynamics library
reserves on each shape, and its lifetime is tied to that shape's.

## Purpose

The dynamics library's collision shapes carry one opaque pointer for the host to use. This
file defines what the engine puts there, and that record is the **join between the solver's
world and the game's world**: given nothing but a contact, everything the engine needs to
decide what the contact means is one dereference away.

That is the single most important structural fact in this chapter. Contact handling runs
thousands of times per step inside a callback that cannot afford a lookup, so every question
it asks — which material, which object, which bone, does this object want a callback, what
did this shape touch last step — is answered from this record.

## State

```text
RECORD GeomUserData                  # attached to every collision shape in the world
  ph_object        : PhysicsObject       # the simulated object this shape belongs to
  ph_ref_object    : ShellHolder         # the GAME object that owns it
  material         : int (16-bit)        # this shape's own surface material
  tri_material     : int (16-bit)        # material of the last static triangle touched
  bone_id          : int (16-bit)        # which skeleton bone this shape follows
  element_position : int (16-bit)        # index of the shell element owning it

  callback         : ContactCallback     # fires for contacts against STATIC geometry
  object_callbacks : list<ObjectContactCallback>   # fire for contacts against objects
  callback_data    : opaque              # caller's context for both

  # --- motion history, for the swept-collision and degeneracy machinery ---
  last_pos         : vector              # shape centre at the end of the previous step;
                                         # negative-infinity means "no history"
  pushing_neg      : bool                # last step this shape was pushed by a triangle
  pushing_b_neg    : bool                #   whose normal faced away from it
  neg_tri          : Triangle            # the offending triangles, remembered so the
  b_neg_tri        : Triangle            #   same decision is made consistently next step
  b_static_colide  : bool                # may this shape collide with the world at all

  # --- the static-triangle cache ---
  cashed_tries     : list<int>           # indices into the level's triangle soup
  last_aabb_pos    : vector              # the box the cache was gathered for
  last_aabb_size   : vector              # zero size means "cache invalid"
```

**Invariants** — a shape either has this record for its whole life or never has one;
`ph_object` and `ph_ref_object` are non-null for every shape belonging to a simulated
object, and both null for the level's static mesh. The triangle cache is valid only while
the shape's swept bounding box stays inside the box the cache was gathered for; any motion
beyond that must clear it.

## the triangle cache

**Contract** — rather than query the static collision database on every step, a moving shape
remembers the triangles its neighbourhood contains and re-queries only when it leaves that
neighbourhood.

**Notes** — this is the chapter's main performance decision and the reason the mesh collider
can afford to be exact. The trade is memory and staleness: a level's geometry never changes,
so a cached triangle list cannot go wrong, only get bigger than needed. The debug build
keeps a global count of cached triangle references precisely to catch the cache growing
without bound, which is what happens when the invalidation box is computed wrongly.

## `ObjectContactCallback` chain

**Contract** — a shape may have several object-contact callbacks, held as a chain and called
in the order they were added, each seeing the contact the previous one may have modified.
Adding a callback twice is a programming error; removing one that is not present is
harmless.

**Notes** — the chain exists because callbacks come from different owners that do not know
about each other: the shell installs one, the camera-collision path installs another on one
specific shape, a script may install a third. A single slot would have them silently
overwrite each other, which is precisely the bug the chain replaced.

Every callback receives a mutable *do-collide* flag and a mutable contact. Setting the flag
false suppresses the contact entirely; mutating the contact changes the friction, bounce or
softness the solver will use. This is the mechanism by which all game-specific physics
behaviour in this chapter is expressed — there is no other hook.

## shape-transform unwrapping

**Contract** — a shape attached to a body at an offset is wrapped in a transform shape, so a
contact may name either the wrapper or the inner shape. `retrieveGeom` unwraps one level;
`retrieveGeomUserData` and `retrieveRefObject` unwrap and then read the record.

**Notes** — the user data lives on the **inner** shape, not the wrapper, and forgetting to
unwrap is the single most common mistake in this chapter's callbacks. A rebuild that
represents a shape's local offset as a field rather than as a wrapper shape deletes this
whole class of bug.

## creation and destruction

**Contract** — creating the record attaches it to a shape with every field at its neutral
value: no history, no callbacks, no object, material zero, collision with static geometry
*enabled*. Destroying it releases the callback chain, deducts its cached triangles from the
debug total, and clears the shape's pointer.

**Notes** — the neutral value of the motion history is negative infinity in every component,
chosen so that the first step's "did this shape move far?" test always answers yes and the
swept path runs at least once.
