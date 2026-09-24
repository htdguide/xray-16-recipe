# src/xrEngine/IObjectPhysicsCollision.h

> The read-only window through which a non-physics module asks an object for its physical body.

**Needs** — [`IPhysicsShell.h`](IPhysicsShell.h.md)
**Used by** — [`dx113DFluidObstacles.cpp`](../Layers/xrRenderDX11/3DFluid/dx113DFluidObstacles.cpp.md) · [`PhysicsShellHolder.cpp`](../xrGame/PhysicsShellHolder.cpp.md) · [`PhysicsShellHolder.h`](../xrGame/PhysicsShellHolder.h.md)
**Tier floor** — T3: two accessors, no data

## Purpose

The engine and the renderer sometimes need an object's rigid-body pose — to place a
wallmark, to trace a ray against a ragdoll, to draw a debug shape — without linking
against the physics module. This interface is the minimum an object exposes for that:
its shell, and (historically) a single character element.

It is a separate file because it is the *narrow* half of the physics dependency, and
keeping it narrow is what lets the engine → physics edge be broken at an interface (see
the cycle note in the system requirements).

## State

`Stateless.`

## `IObjectPhysicsCollision`

**Contract** — both accessors return read-only handles and may return nothing when the
object has no physical representation (most objects do not). Neither allocates, neither
blocks. The returned handles are valid only until the object's physics is rebuilt or the
object is destroyed, so callers use them within the frame.

```text
INTERFACE ObjectPhysicsCollision
  physics_shell()     -> optional<PhysicsShell>     # read-only
  physics_character() -> optional<PhysicsElement>   # deprecated: superseded by the shell
```

**Notes** — `physics_character` is marked deprecated in the source and survives only
because call sites still read it. A rebuild should expose the shell alone and let callers
reach element zero through it.
