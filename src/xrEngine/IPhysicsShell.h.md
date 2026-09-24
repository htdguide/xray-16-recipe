# src/xrEngine/IPhysicsShell.h

> The read-only view of a rigid-body assembly: its pose, its elements, their velocities and shapes.

**Needs** — [`IPhysicsGeometry.h`](IPhysicsGeometry.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`dx113DFluidObstacles.cpp`](../Layers/xrRenderDX11/3DFluid/dx113DFluidObstacles.cpp.md) · [`dx113DFluidObstacles.h`](../Layers/xrRenderDX11/3DFluid/dx113DFluidObstacles.h.md) · [`IObjectPhysicsCollision.h`](IObjectPhysicsCollision.h.md) · [`PHCharacter.h`](../xrPhysics/PHCharacter.h.md) · [`PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md)
**Tier floor** — T3: pure accessors over data the physics module owns

## Purpose

A *shell* is the physical representation of one object — a crate is one element, a ragdoll
is a dozen elements joined at the bones. This file declares what the rest of the engine
may read from a shell without linking against the physics module: poses, velocities, mass
centres and shapes, and nothing that mutates.

Declaring it here rather than in the physics module is what makes the engine ⇄ physics
cycle legal: the engine owns the interface, physics provides the adapter.

## State

`Stateless.`

## `IPhysicsElement`

**Contract** — one rigid body. `xform` is its current world transform, valid between
physics steps. Velocities are world-space, linear in units per second and angular in
radians per second. `mass_center` is world-space and is *not* generally the transform's
origin — a body's origin is its authored pivot, its mass centre is where the solver
actually integrates, and code that places effects (smoke, sparks, a grab point) wants the
latter. `get_box` returns an axis-aligned extent and centre for the whole element, which
is the union of its shapes. Geometry is indexed from zero to `number_of_geoms` exclusive.

```text
INTERFACE PhysicsElement
  xform()             -> matrix4                 # world pose
  get_linear_vel()    -> vector3                 # world space
  get_angular_vel()   -> vector3                 # world space, radians/sec
  get_box()           -> (half_extents, center : vector3)
  mass_center()       -> vector3                 # world space; not the xform origin
  number_of_geoms()   -> int (16-bit)
  geometry(index)     -> PhysicsGeometry
```

**Invariants** — `geometry(i)` is defined for every `i < number_of_geoms()`; the count
does not change while the shell exists.

## `IPhysicsShell`

**Contract** — an assembly of elements. `xform` is the shell's root pose — for a
single-element shell it equals element zero's, for a ragdoll it is the root bone's.
Elements are indexed from zero; the index is stable for the life of the shell and matches
the order the shell was built in, which is the order the skeleton's bones were walked, so
callers that also hold the skeleton can pair the two by index.

```text
INTERFACE PhysicsShell
  xform()               -> matrix4
  element(index)        -> PhysicsElement
  get_elements_number() -> int (16-bit)
```

**Invariants** — element count is fixed at construction; element index ↔ bone index
correspondence is established when the shell is built and relied upon by animation
blending.
