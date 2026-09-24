# src/xrPhysics/PhysicsShellAnimatorBoneData.h

> One controlled bone: the body being dragged and the constraint doing the dragging.

**Needs** — [`PHShell.h`](PHShell.h.md) · [`PhysicsShellAnimator.h`](PhysicsShellAnimator.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PhysicsShellAnimator.cpp`](PhysicsShellAnimator.cpp.md) · [`PhysicsShellAnimator.h`](PhysicsShellAnimator.h.md)
**Tier floor** — T2: a pair of handles.

## Purpose

A record, split into its own file only because the animator's header would otherwise need the
shell's internals. The split is arbitrary and a rebuild should fold it into the animator.

## State

```text
RECORD ControlledBone
  element    : Element        # the rigid body this bone drives
  constraint : Joint          # a FIXED constraint between that body and the world,
                              # whose target pose is rewritten every frame
  # invariant: the constraint is registered with the shell's active island for as
  # long as this record exists, and is unregistered before it is destroyed
```

**Notes** — the constraint is a *fixed* one — zero degrees of freedom — rather than a motor or
a spring. That is the load-bearing choice: re-aiming a rigid weld every frame makes the body
follow the animation exactly while still participating in collision, whereas a spring would
lag and a kinematic placement would not collide at all. The cost is that a body wedged against
the world fights the weld, which the solver resolves by softening it; see
[`PhysicsShellAnimator.cpp`](PhysicsShellAnimator.cpp.md).
