# src/xrPhysics/dcylinder

> The cylinder primitive, added to a dynamics library that does not have one.

Part of chapter 16, [`src/xrPhysics`](../README.md).

## What this module is responsible for

The [dynamics seam](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) demands a
sphere, a box, a capsule and a cylinder. The library the original uses supplies the first
three. This directory supplies the fourth: a new shape kind registered with the library at
run time, together with the narrow-phase routines that collide it against a box, a sphere,
another cylinder, a plane and a ray.

It is not an optional nicety. Every character in the game stands on a cylinder — the
capsule's flat-bottomed cousin — because a capsule's rounded cap makes a character slide off
every edge it stands near and wobble on flat ground. Vehicle wheels are cylinders for the
obvious reason. A rebuild whose dynamics library already offers a cylinder can delete this
directory outright and lose nothing; a rebuild whose library does not must reproduce it, and
the two pages here are the specification.

## Where it sits

It rests on the dynamics library's shape-registration mechanism and on the module's own math
helpers, and it is depended on by the character controllers, the vehicle code and — most
intricately — by [`../tri-colliderknoopc`](../tri-colliderknoopc/README.md), which must
collide a cylinder against the level's triangle soup. That dependency runs both ways at the
source level: the cylinder-versus-triangle routine lives next door and is pulled in from
here, which is a build-order artefact rather than a design.

## The load-bearing idea

**A shape kind is a small vtable and a blob of bytes.** The library is told the size of the
shape's own data, a function that reports its bounds, a function that returns the collision
routine for a given other kind, and a destructor. Everything else — the shape's parameters,
its transform, its body — belongs to the library. This is the mechanism the
[custom triangle collider](../tri-colliderknoopc/README.md) uses as well, and understanding
it once covers both.

**Every collision routine here is the same technique.** Enumerate candidate separating axes
(the cylinder's own axis, the other shape's axes, and the cross products that bound the rim);
find the one of least penetration; then synthesize a contact manifold appropriate to the
feature pair that axis implies — a point for a corner, a pair for an edge, a polygon for a
face. Nothing here is a general convex solver, and the manifolds are built by cases because
the cases are few and known.

## Twins

| File | Role |
|---|---|
| [`dCylinder.h`](dCylinder.h.md) | Declares the shape kind, its parameter accessors and the collision entry points. |
| [`dCylinder.cpp`](dCylinder.cpp.md) | The shape kind's registration and bounds, and the five narrow-phase routines. |
