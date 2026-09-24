# src/xrGame/PHShellCreator.cpp

> Builds a rigid-body shell for an object straight from its skeleton, with no per-object authoring.

**Needs** — [`PHShellCreator.h`](PHShellCreator.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`GameObject.h`](GameObject.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: assembling a body from model data

## Purpose

The default answer to "give this object a physical body". Most objects in the game have no
authored physics at all — their shapes, masses and joints are carried in the model's own
skeleton, where the artist placed them alongside the bones. This is the one-shot builder
that turns that into a live shell.

It exists as a separate strategy object because some kinds of object need a different one: a
character's body is a capsule driven by a controller, a vehicle's is an authored assembly.
Those supply their own builder; everything else gets this.

## State

`Stateless.` The shell it builds belongs to the object.

## `CreatePhysicsShell`

**Contract** — builds and installs a shell on the object this strategy is mixed into. Does
nothing when the object has no visual (nothing to derive shapes from) or already has a shell
— so it is safe to call more than once, which matters because several spawn paths reach it.

```text
FUNCTION create_physics_shell()
  owner = this object as a physics shell holder
  IF owner has no visual         -> RETURN
  IF owner already has a shell   -> RETURN
  verify the model actually carries physics data          # a hard failure if not
  shell = new empty shell
  shell.build_from_skeleton(owner's skeleton, root bone)  # shapes, masses, joints
  shell.reference_object = owner                          # so hits find their way back
  shell.transform = owner's current placement
  shell.air_resistance = (linear 0.001, angular 0.02)
```

**Invariants** — the shell's transform is taken from the object *after* building and not
before: the builder produces a shell in the model's own space, and placing it is a separate
step. Reversing them puts the object at the origin.

The reference back to the owner is what lets a collision or a hit arriving at a body find the
game object it belongs to. Without it the physics world's results are anonymous.

**Notes** — the two air-resistance coefficients, one linear and one angular, are hard-coded
here for every object built this way. They are small damping terms that stop a free body
drifting forever, and nothing in the tree tunes them per object. A rebuild is free to make
them configuration, but should keep them non-zero: the rigid-body solver alone does not
settle.

A commented-out inertia-smoothing call suggests the built inertia tensors were once
blended toward a sphere to stabilize thin parts. It is off, and the consequence is that a
flat or thin fragment built this way can behave erratically.
