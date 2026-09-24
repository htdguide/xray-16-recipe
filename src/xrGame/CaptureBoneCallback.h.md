# src/xrGame/CaptureBoneCallback.h

> The interface a caller implements to decide which bone of a ragdoll may be grabbed.

**Needs** — [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHMovementControl.cpp`](PHMovementControl.cpp.md) · [`group_state_eat_drag_inline.h`](ai/monsters/group_states/group_state_eat_drag_inline.h.md)
**Tier floor** — T3: an interface declaration

## Purpose

When something in the game grabs a physical object — the player dragging a body, a creature
seizing a victim — it must pick *which* rigid body of that object to attach to. The physics
layer offers a nearest-to-a-point search; this interface is how the game filters that
search's candidates.

It is substantive despite being three lines, because what it demands of an implementor *is*
the contract: given a bone, answer whether it is an acceptable grab point.

## `CPHCaptureBoneCallback`

**Contract** — extends the physics layer's nearest-to-point search predicate. The
implementor supplies one answer:

```text
FUNCTION accepts(bone_index) -> bool
```

and the interface adapts the physics layer's element-based query to it by taking the
element's own bone index. That adaptation is the only code in the file, and its purpose is
to let the game reason in *bones* — which it names, animates and assigns damage to — while
the physics layer reasons in *elements*.

**Invariants** — the two overloads must agree: filtering an element must be exactly
filtering its bone. A rebuild that lets an element carry more than one bone breaks the
identity this interface assumes.
