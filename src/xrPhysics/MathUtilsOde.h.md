# src/xrPhysics/MathUtilsOde.h

> The numerically careful normalize, the velocity clamp, and the restitution
> helpers that sit right on the dynamics library's edge.

**Needs** — [`MathUtils.h`](MathUtils.h.md) · [`ode_redefine.h`](ode_redefine.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`Geometry.h`](Geometry.h.md) · [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md) · [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHContactBodyEffector.cpp`](PHContactBodyEffector.cpp.md) · [`PHDisabling.cpp`](PHDisabling.cpp.md) · [`PHDynamicData.h`](PHDynamicData.h.md) · [`PHJointDestroyInfo.cpp`](PHJointDestroyInfo.cpp.md) · [`PHValideValues.h`](PHValideValues.h.md) · [`PhysicsExternalCommon.cpp`](PhysicsExternalCommon.cpp.md) · [`collisiondamagereceiver.cpp`](collisiondamagereceiver.cpp.md) · [`dTriBox.h`](tri-colliderknoopc/dTriBox.h.md) · [`dTriColliderMath.h`](tri-colliderknoopc/dTriColliderMath.h.md)
**Tier floor** — T1: it exists to work directly on the solver's own float layout and to
compensate for the precision of the solver's own square root.

## Purpose

[`MathUtils.h`](MathUtils.h.md) holds math that stands on its own; this file holds math that
only makes sense next to the solver. Splitting them is not arbitrary — it keeps the
solver-free half of the module compilable without the dynamics library, which is what lets
the character controller's geometry be reasoned about independently.

## `accurate_normalize`

**Contract** — normalize a three-component direction in place, correctly for vectors whose
squared magnitude underflows. A zero vector becomes a unit vector along the first axis
rather than producing a non-number.

**Invariants** — never produces a non-finite component, for any finite input including zero.

```text
FUNCTION accurate_normalize(v)
  sq = v.x^2 + v.y^2 + v.z^2
  IF sq > tiny_threshold                 # the overwhelmingly common path
    v = v * reciprocal_sqrt(sq)
    RETURN
  # sq underflowed. Divide through by the LARGEST component first, so the
  # squares that follow are all at most one and cannot underflow again.
  largest = index of component with greatest magnitude
  IF v[largest] == 0
    v = (1, 0, 0)                        # a whole zero vector: pick a direction
    RETURN
  divide the two other components by |v[largest]|
  l = reciprocal_sqrt(other1^2 + other2^2 + 1)
  set the two others to other * l, and v[largest] to l with its original sign
```

**Notes** — the threshold is the square root of the smallest normal single-precision value,
which is exactly the point below which squaring loses everything. The reason this matters at
all: contact normals arrive from the mesh collider as cross products of triangle edges, and
a sliver triangle produces a cross product small enough to underflow. Without this path, one
degenerate triangle in a level puts a non-number into a body's velocity and the object
disappears. A rebuild whose language has a robust normalize already may use it; a rebuild
that writes the naive one will ship this bug.

Choosing the largest component rather than any non-zero one is not an optimization — it is
what guarantees the remaining squares are bounded by one.

## `dVectorLimit`

**Contract** — clamp a vector to a maximum magnitude, writing the result to a separate
output and reporting whether clamping occurred. The direction is preserved exactly.

**Notes** — the *reporting* is what callers use: the velocity limiters in this chapter need
to know whether a clamp happened so they can correct the body's position to match the
velocity they just changed, and doing that unconditionally would introduce drift.

## `E_NL` / `E_NlS`

**Contract** — declared here, defined in [`Physics.cpp`](Physics.cpp.md). Given two bodies
(or one body and static geometry) and a contact normal, report the effective mass along that
normal — the reduced mass of the pair as seen by an impulse along the contact. The signed
variant takes which side of the contact the body is on.

**Notes** — this is the quantity that turns a contact into an impact *energy*, and it is why
a heavy crate striking a character hurts more than a light one at the same speed. A rebuild
computing collision damage from closing speed alone will get the mass dependence wrong.

## `dVectorInterpolate`

**Contract** — linear blend between two directions into a separate output, leaving both
inputs untouched. It exists only because the corresponding routine in
[`MathUtils.h`](MathUtils.h.md) destroys its second argument; this is the non-destructive
wrapper, and a rebuild needs only the non-destructive form.
