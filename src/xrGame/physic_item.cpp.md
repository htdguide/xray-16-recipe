# src/xrGame/physic_item.cpp

> The simplest physical object: a rigid body while loose in the world, nothing at all while carried, and two shell shapes derived from the model's own bounding box.

**Needs** — [`physic_item.h`](physic_item.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`Include/xrRender/RenderVisual.h`](../Include/xrRender/RenderVisual.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: rigid-body lifecycle tied to ownership transitions

## Purpose

Everything a dropped object needs and nothing more. The file's whole content is one idea —
**a physical item is simulated exactly when it has no owner** — applied consistently across
the entity lifecycle, plus two procedurally-derived collision shapes for items that have no
authored physics model.

## State

```text
RECORD PhysicItem                    # a physics-shell holder
  shell              : optional<physics shell>
  ready_to_destroy   : bool          # set by subclasses that are mid-teardown
```

**Invariants** — the shell exists only while the item has no parent. Every transition between
owned and loose creates or destroys it, and the per-frame update reads it only when there is
no parent. A rebuild that lets a carried item keep a live body will have carried weapons
colliding with their owner.

## `net_Spawn`

**Contract** — after the base spawn, forces the model's bone transforms to be recomputed, then
builds a physics shell **only if the server record says the item has no parent**. Makes the
item visible and enabled. Fails if the base fails.

```text
FUNCTION net_Spawn(server record)
  IF base spawn fails THEN RETURN failure
  invalidate and recompute the model's bone transforms
  IF the record's parent identifier is the "no parent" sentinel
    IF there is no shell yet THEN setup_physic_shell()
  make visible; make enabled
  RETURN success
```

**Invariants** — the bone recomputation must happen **before** the shell is built. The shell's
element positions are read from the posed skeleton, and a skeleton still holding its
bind pose produces a body in the wrong place. This ordering appears three times in this file
— on spawn, on activation and on setup — and is the one thing most easily lost in a rebuild,
because in each case it looks like a redundant refresh.

The parent test is against the *server record*, not the client state, because at spawn time
the client-side parent link has not yet been established. An item spawned into someone's
inventory must never get a body even for one frame.

## `OnH_B_Independent` / `OnH_B_Chield`

**Contract** — the two ownership transitions, and the symmetry between them is the file's
thesis.

```text
FUNCTION on_becoming_independent(just_before_destroy)
  base handling
  IF already committed to destruction THEN RETURN
  make visible; make enabled
  IF NOT just_before_destroy THEN activate_physic_shell()

FUNCTION on_becoming_owned()
  base handling
  make invisible; make disabled
  deactivate the physics shell
```

**Invariants** — becoming independent *just before destruction* must not build a body. The
item is about to cease existing and a freshly created rigid body would be registered with the
physics world after the teardown sweep that would have removed it — the engine-wide invariant
that a destroyed entity is unreferenced by the physics world before its memory is released.
The same reasoning covers the already-committed flag.

Visibility and simulation are switched together in both directions. A carried item is neither
drawn nor updated; the thing the player sees in their hands is a separate in-hand
representation, not this object.

## `UpdateCL`

**Contract** — each frame, if the item has no parent and its shell exists and is active, take
the shell's **interpolated** global transform as the item's transform. Then run the base
update.

**Invariants** — interpolated, not raw. The physics world steps at its own rate, which is not
the frame rate; reading the raw body transform makes small objects visibly stutter. The
interpolation is the physics layer's, and asking for it is the whole decision here.

The order matters: the transform is taken *before* the base update, so everything downstream —
attachments, sounds, bone computation — sees this frame's position.

## `activate_physic_shell` / `setup_physic_shell`

**Contract** — `activate_physic_shell` snaps the item's transform to its **parent's** before
building the body, then recomputes bones. `setup_physic_shell` builds from the item's own
transform, then recomputes bones.

**Invariants** — the transform snap in the activation path is what makes a dropped item appear
where its owner was rather than where it was last drawn. A carried item's own transform is
stale — it has not been updated while carried, since it was disabled — so it must be taken
from the parent at the moment of release. A rebuild that keeps carried items' transforms
current does not need the snap, but then must ensure the transform is the *hand's*, not the
owner's origin.

Both recompute bones after building, for the same reason the spawn path does it before: the
shell's construction may move the model, and anything reading bone positions this frame must
see the result.

## `create_box_physic_shell`

**Contract** — builds a single-element shell containing one axis-aligned box taken from the
model's own bounding volume, with a fixed density.

```text
FUNCTION create_box_physic_shell()
  box := the model's collision bounding box, with identity rotation
  element := a new physics element containing that box
  shell := a new shell containing that element
  shell density := 2000
```

**Invariants** — the density is 2000 in the physics library's units, which for the box volumes
involved yields masses in the range small props are tuned for. No derivation for the exact
figure is recoverable; it is a single value shared by both procedural shapes, so items built
this way are all made of the same notional substance.

## `create_box2sphere_physic_shell`

**Contract** — builds a single-element shell containing the model's box **plus two spheres
along its longest axis**, one large at the far end and one small at the near end, with the two
shorter box extents halved. Sets air resistance on the shell.

```text
FUNCTION create_box2sphere_physic_shell()
  box := the model's collision bounding box, identity rotation
  longest := whichever of the three half-extents is largest
  axis := the box's unit vector along `longest`, scaled by that half-extent
  radius := the smaller of the two remaining half-extents
  halve the two remaining half-extents           # the box shrinks; the spheres take over
  sphere_far  := centre + axis,  radius = radius * sqrt(2)
  sphere_near := centre - axis,  radius = radius / 2
  element := box + sphere_far + sphere_near
  shell := a new shell containing that element
  shell density := 2000
  enable air resistance
```

**Invariants** — this is a *capsule approximation built by hand*: a thin central box with a
fat sphere at one end and a small one at the other, which tumbles and rolls the way a thrown
object should rather than catching on edges the way a box does. The asymmetry between the two
sphere radii — one enlarged by the square root of two, the other halved — puts the shape's
mass and its rolling contact toward one end, so a thrown object lands nose-first and settles
rather than spinning. This is the bolt's shape.

The square root of two is the ratio that makes a sphere circumscribe a square cross-section of
the given half-extent, so the large sphere fully contains the box's cross-section at that end
— nothing of the box protrudes from it. The near sphere's half radius has no such derivation
and is a tuning choice.

Halving the two shorter extents *after* the radius is taken from them means the spheres are
sized to the original cross-section and the box is then shrunk inside them, so the box
contributes only the central span.

**Notes** — air resistance is enabled on this shape and not on the plain box. A thrown item
needs it to fly on a believable arc; a dropped crate does not.

The selection of the longest axis is written as a three-way comparison macro, which is
incidental; the decision it encodes — pick the longest axis, take the radius from the smaller
of the other two — is not.

## `create_physic_shell`

**Contract** — delegates to the base, which builds a shell from the model's *authored* physics
description.

**Notes** — the procedural box call is present and commented out. So the default for a physical
item is the authored shell, and the two procedural shapes on this page are used only by
subclasses that override this and call them explicitly. A rebuild should read the two
procedural builders as *available shapes*, not as the default path.
