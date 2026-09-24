# src/xrPhysics/PhysicsExternalCommon.h

> The vocabulary the physics module and the game layer share at their boundary: the four
> callback shapes, the oriented-box query, and the movement-restriction classes.

**Needs** — [`xrPhysics.h`](xrPhysics.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`PhysicsExternalCommon.cpp`](PhysicsExternalCommon.cpp.md) · [`xrCDB/xrCDB.h`](../xrCDB/xrCDB.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHMovementControl.h`](../xrGame/PHMovementControl.h.md) · [`IPHStaticGeomShell.h`](IPHStaticGeomShell.h.md) · [`IPHWorld.h`](IPHWorld.h.md) · [`PHActorCharacter.h`](PHActorCharacter.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`PhysicsExternalCommon.cpp`](PhysicsExternalCommon.cpp.md) · [`PhysicsShell.h`](PhysicsShell.h.md)
**Tier floor** — T2: four function shapes and one geometric routine. Only the contact record
they pass by reference is T1, and that record belongs to the dynamics seam.

## Purpose

Everything the game layer needs in order to *hear about* physics without linking against the
dynamics library's headers lives here. That is deliberately little: four callback shapes, a
box query, and one enumeration. A rebuild that hides the dynamics library behind an interface
will find this file is the interface's other half — the part that flows outward.

## Stateless.

## the four callback shapes

**Contract** — a caller registers one of these and the physics step calls it back from inside
collision detection. All four are invoked *while a step is in progress*, which is the single
most important fact about them: a callback that mutates the world, creates an object or
destroys one corrupts the step. The world exposes a "processing" flag precisely so this class
of bug can be asserted against ([`IPHWorld.h`](IPHWorld.h.md)).

```text
# fires once per generated contact against a triangle of the static collision database
CALLBACK on_static_contact(triangle, contact)
  # the receiver may read the triangle's material and the contact's position and
  # normal, and typically leaves a decal or plays an impact sound

# fires once per candidate contact between two collision shapes, BEFORE the
# contact is handed to the solver
CALLBACK on_object_contact(inout accept : bool,
                           this_side_is_first : bool,
                           inout contact,
                           material_of_first, material_of_second)
  # the receiver may reject the contact outright (accept := false), retune its
  # friction and bounce from the two materials, or record damage from it

# fires once per bone whose transform the physics owns, when the skeleton is
# evaluated for rendering
CALLBACK on_bone(bone_instance)

# fires once per completed step with the wall-clock bracket of that step
CALLBACK on_step_time(start_ms, end_ms)
```

**Invariants** — `this_side_is_first` tells the receiver *which* of the two shapes is the one
it registered on. Every contact is presented once, not twice, so a receiver that wants a
symmetric effect must produce it from this flag rather than expecting a second call with the
shapes swapped. Getting this wrong halves or doubles every collision effect in the game.

**Notes** — the object-contact callback's `accept` is an out-parameter rather than a return
value in the original because the chain form (several receivers on one shape) needs each link
to be able to veto without ending the chain. The load-bearing decision is that rejection is
sticky and tuning is cumulative, not that the flag is passed by reference.

## `contact_shot_mark_effect_params`

**Contract** — declared here, implemented in
[`PhysicsExternalCommon.cpp`](PhysicsExternalCommon.cpp.md).

## oriented bounding box of a physical thing

**Contract** — given anything that can report its extent along an arbitrary axis (a shape, an
element, a whole shell) and an orientation matrix, produce the size and centre of the box in
*that* orientation which just contains it.

```text
FUNCTION oriented_box(thing, orientation) -> (size, centre)
  centre := (0,0,0)
  FOR EACH i IN 0..2
    axis := row i of orientation
    (lo, hi) := thing.extent_along(axis)
    size[i]  := hi - lo
    centre   := centre + axis * ((lo + hi) / 2)
  RETURN (size, centre)
```

**Notes** — this is *not* an axis-aligned box of axis-aligned boxes. Each axis is queried
independently against the true shapes, so the result is the tightest box in the given frame.
It exists as one routine over an abstract "can report its extent" because the same code must
serve a single shape, one rigid element and an entire multi-body shell, all of which answer
the extent query differently. In the original that genericity is a template; in a rebuild it
is one function over one small interface.

## `ERestrictionType`

**Contract** — the movement-restriction class an entity belongs to: human-sized, small human,
medium monster, none, or the player. A restrictor volume admits or refuses an entity by this
class, so it is the coarse body-size taxonomy the whole navigation and restriction layer
shares.

**Notes** — it lives in the physics module's outward-facing header rather than with the
restrictors because the character controller reads it when sizing its capsule. That placement
is an accident of history; a rebuild should put it with the entity definitions and have the
character controller import it.
