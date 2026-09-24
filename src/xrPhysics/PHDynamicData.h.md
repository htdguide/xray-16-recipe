# src/xrPhysics/PHDynamicData.h

> The conversion between the engine's row-major transform and the dynamics library's column-major rotation — plus an abandoned hierarchical pose cache.

**Needs** — [`PHInterpolation.h`](PHInterpolation.h.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · [`PHDynamicData.cpp`](PHDynamicData.cpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`Geometry.cpp`](Geometry.cpp.md) · [`PHActivationShape.cpp`](PHActivationShape.cpp.md) · [`PHCharacter.cpp`](PHCharacter.cpp.md) · [`PHDynamicData.cpp`](PHDynamicData.cpp.md) · [`PHElement.cpp`](PHElement.cpp.md) · [`PHJoint.cpp`](PHJoint.cpp.md) · [`Physics.cpp`](Physics.cpp.md) · [`PhysicsShellAnimator.cpp`](PhysicsShellAnimator.cpp.md)
**Tier floor** — T1: the whole live content of this file is a byte-level reinterpretation of one matrix layout as another.

## Purpose

Two libraries, two matrix conventions. The engine stores a transform as three basis rows plus a
translation row; the dynamics library stores a rotation as three basis *columns* in a padded
twelve-element array. Every crossing between them goes through the four conversions here, and they
are the only part of this file that is live.

The rest of the file — the hierarchical pose cache that gives the type its name — is compiled out
in its entirety. See the note below.

## The conversions

**Contract** — pure, allocation-free, and exact. Each is a transpose plus, where relevant, a
translation copy.

```text
FUNCTION solver_to_engine(rotation, position) -> transform
  # the library's rotation is this transform's rotation, transposed
  transform.basis    := transpose(rotation)
  transform.position := position
  transform.homogeneous_row := (0, 0, 0, 1)

FUNCTION solver_rotation_to_engine(rotation) -> transform    # orientation only
FUNCTION engine_to_solver(transform)        -> rotation      # the inverse transpose
FUNCTION engine_basis_to_solver(basis)      -> rotation      # for a bare 3×3
```

**Invariants** — the library's array is twelve elements with a padding slot after every row; the
conversions must skip it. This is the layout dependency that pins the file at T1 — a rebuild that
defines its own body record deletes the padding and most of this file with it.

**Notes** — `solver_to_engine` is implemented as a block copy followed by an in-place transpose
rather than element-by-element. That is a micro-optimisation on a path that runs once per body per
frame; it is not a decision a rebuild must reproduce, but the transpose itself is.

The parallel `GetWorldMX` / `GetTGeomWorldMX` helpers build a body's world transform and a
transformed sub-shape's world transform respectively. The second composes four frames — the
sub-shape's local offset, the body's orientation, the body's position, and the sub-shape's own
rotation — and exists because a shape attached to a body through an offset wrapper has no single
stored world transform to read.

## The dead hierarchy

The declared `PHDynamicData` record — a body with a fixed number of children, each with its own
interpolation, a recorded rest pose, and recursive "compute every child's transform relative to its
parent" operations — has **no live implementation**; the entire body of
[`PHDynamicData.cpp`](PHDynamicData.cpp.md) is disabled.

What it was for is legible from the shape: caching a skeleton's worth of body transforms as a tree,
so a ragdoll's bone poses could be computed once per frame by one recursive walk instead of
per-bone on demand. The engine ended up doing that work in the per-bone callback instead (see
`bones_callback` in [`PHElement.cpp`](PHElement.cpp.md)), which is simpler and interacts correctly
with partial physics-on-animation blending.

A rebuild should implement the conversions and ignore the record. If profiling later shows the
per-bone path is the cost, this is the shape of the fix that was attempted.
