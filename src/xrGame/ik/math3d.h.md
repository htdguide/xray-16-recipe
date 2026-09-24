# src/xrGame/ik/math3d.h

> Declares the solver's own linear algebra — a parallel universe to the engine's, kept
> because the solver was vendored whole.

**Needs** — _(none)_
**Used by** — [`Dof7control.cpp`](Dof7control.cpp.md) · [`Dof7control.h`](Dof7control.h.md) · [`IKLimb.cpp`](IKLimb.cpp.md) · [`eulersolver.cxx`](eulersolver.cxx.md) · [`eulersolver.h`](eulersolver.h.md) · [`math3d.cpp`](math3d.cpp.md)
**Tier floor** — T2. Vector and matrix arithmetic on fixed-size float arrays.

## Purpose

Declares the surface implemented in [`math3d.cpp`](math3d.cpp.md), and carries the small
componentwise operations inline because they are one line each.

The whole file is **replaceable by the engine's own math layer**
([`src/utils/xrMiscMath`](../../utils/xrMiscMath/README.md)) and a rebuild should replace
it. It exists because the inverse-kinematics solver arrived from outside with its own
types, and reconciling them was never done. The one thing that does not transfer for free
is the *convention*: see the implementation twin.

## Exported units

Two type names, and everything below is expressed in them:

- a 4×4 transform, stored row-major, applied to **row** vectors on the left — `v · M`;
- a quaternion, stored scalar-part-first.

Composition and inversion: general 4×4 multiply, a rigid-transform-only multiply, a
rotation-only multiply, rigid-transform inverse, rotation-only inverse.

Applying a transform: to a point, and to a direction.

Rotation construction and recovery: from axis and angle (two entry points, one tuned for
the principal axes), from a principal axis by name, the derivative of a principal-axis
rotation, and recovery of axis and angle from a matrix.

Quaternions: to and from a rotation matrix, to and from axis-angle, 4-vector normalize.

Vector helpers (inline): scale, add, subtract, cross, dot, normalize returning the prior
length, length.

Translation access: read and write the transform's position, by components or as a vector,
plus a read that returns only its magnitude — which is how a link's length is taken.

Geometry: project a vector onto another, project a vector onto a plane by its normal,
signed angle between two vectors about an axis, and *find any vector perpendicular to
this one*.

Interpolation: componentwise vector interpolation, and rigid-transform interpolation that
slerps the rotation through its axis-angle form.

Debug printing of a matrix and a vector.
