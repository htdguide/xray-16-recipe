# src/xrPhysics/PHJoint.h

> Declares the one joint type that covers every constraint in the game — five shapes behind one axis-oriented surface — and carries the axis-and-angle extraction that the limit machinery is built on.

**Needs** — [`PHJoint.cpp`](PHJoint.cpp.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md) · [`MathUtils.h`](MathUtils.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHFracture.cpp`](PHFracture.cpp.md) · [`PHJoint.cpp`](PHJoint.cpp.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHShell.h`](PHShell.h.md) · [`PHShellActivate.cpp`](PHShellActivate.cpp.md) · [`PHShellBuildJoint.h`](PHShellBuildJoint.h.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`PhysicsShell.cpp`](PhysicsShell.cpp.md)
**Tier floor** — T1: the declared surface holds solver joint handles and hands raw axis vectors across the dynamics-library boundary.

## Purpose

Declares the surface implemented in [`PHJoint.cpp`](PHJoint.cpp.md), and holds — inline, because
they are used during construction — the four rotation-decomposition helpers that turn a relative
transform between two bodies into an axis and an angle.

The one decision the whole type embodies: **the game never names a joint kind, it names axes.**
Every constraint in the game — a ragdoll's shoulder, a door's hinge, a wheel's suspension, a
sliding hatch — is configured the same way: give it an anchor, give it up to three axes, give each
axis a low and high stop, a spring and damping factor, and optionally a motor force and target
velocity. The concrete constraint is chosen from that description once, at creation. A rebuild that
exposed five different joint APIs would have to change every call site in the game layer.

## State

```text
RECORD Joint
  type            : one of {ball, hinge, hinge2, slider, full_control}
  bone_id         : int              # which skeleton bone this joint animates
  first_element   : Element          # the parent body
  second_element  : Element          # the child body
  first_geom      : optional<Shape>  # which shape of the parent the joint is anchored to
  shell           : Shell
  joint           : solver constraint handle
  joint1          : optional<solver constraint handle>   # see the notes
  back_ref        : a slot to null when this joint dies
  destroy_info    : optional<JointDestroyInfo>           # present only if breakable
  erp, cfm        : real             # the joint's own softness (see PhysicsCommon)
  anchor          : vector
  anchor_frame    : one of {first, second, global}
  axes            : list<Axis>       # 0, 1, 2 or 3 entries depending on the type
  active          : bool

RECORD Axis
  high, low       : real      # stops; default unbounded
  zero            : real      # the angle the two bodies were at when the stops were authored
  erp, cfm        : real      # the STOP's softness, distinct from the joint's
  frame           : one of {first, second, global}   # what `direction` is expressed in
  force           : real      # motor's maximum force; 0 means no motor
  velocity        : real      # motor's target rate
  direction       : vector    # default (0,0,1)
```

**Invariants** — the axis count is fixed by the type at construction and never changes: ball has
none, hinge one, hinge-2 and slider two, full-control three. Every axis-indexed call clamps its
index into that range rather than failing, so a caller may always ask for axis 2 and get something
sensible.

`zero` is captured when the stops are set, not when the joint is created; it is the reference angle
the authored stops are relative to. Setting the stops on a joint whose bodies are not in their
authored rest pose therefore silently shifts the whole range.

## `joint1` — why some joints are two constraints

**Contract** — three of the five shapes are built from *two* solver constraints rather than one,
and every operation on them dispatches to whichever one owns the axis in question.

| Type | constraint | second constraint | axes |
|---|---|---|---|
| ball | ball-and-socket | — | none: free rotation |
| hinge | hinge | — | one, with stops and a motor |
| hinge2 | hinge-2 | — | two: steering and spin, plus a sprung suspension along the first |
| slider | slider (translation) | an angular motor, one axis | translation on axis 0, rotation on axis 1 |
| full_control | ball-and-socket | an angular motor, three axes | three independent rotation limits |

**Notes** — the reason is that the dynamics library offers positional constraints with *at most* one
or two limited degrees of freedom, and an angular motor as a separate mechanism that can limit
three. A three-axis limited rotation — which is what a ragdoll shoulder is — must therefore be a
ball-and-socket holding the position plus a motor holding the orientation. A rebuild against a
library with a native six-degree-of-freedom joint collapses all five of these into one, and that is
the right thing to do; what must survive is the *axis description*, because that is what the game
data holds.

## the axis extraction helpers

These are the mathematical content of the header, and they exist because the joint's stops are
authored as angles about named axes while the bodies' relationship is a transform.

**`own_axis`** — given a rotation, recovers the axis it rotates about, by solving for the
eigenvector with eigenvalue one. Special-cases two degeneracies: a rotation that is the identity
about the first coordinate axis, and a rotation in one coordinate plane.

**Notes** — the closed-form solve is the interesting decision. A rebuild is free to use a quaternion
logarithm or any other axis extraction; the requirement is only that the *sign* convention matches,
because the stops are signed. The degenerate branches exist because the general solve divides by a
quantity that vanishes when the rotation has no component out of a plane.

**`own_axis_angle`** — the axis, plus the angle turned about it. Builds a pair of orthonormal vectors
perpendicular to the axis, rotates one of them, and reads the angle off as the arc-cosine of the
dot product with the sign taken from the other. Falls back to a fixed perpendicular when the axis
lies along the first coordinate direction.

**`axis_angleA` / `axis_angleB`** — the angle about a *given* axis, which is what the limit code
actually needs. They differ in one thing: **A** transforms the axis by the rotation before building
the perpendicular pair, **B** does not. So **A** measures the angle in the *rotated* frame and **B**
in the original. The live path uses **A**; **B** is present, correct, and called nowhere.

```text
FUNCTION angle_about(rotation, axis) -> real
  # two orthonormal vectors spanning the plane perpendicular to `axis`
  IF axis is not along the first coordinate direction THEN
    ort1 := (0, -axis.z, axis.y)
  ELSE
    ort1 := (0, 1, 0)
  ort2 := normalize(axis × ort1) ; ort1 := normalize(ort1)

  turned := rotation applied to ort1
  # project the turned vector back INTO the plane before measuring
  p1 := ort1 · turned ; p2 := ort2 · turned
  IF p1 = 0 AND p2 = 0 THEN RETURN 0        # the turned vector is along the axis: no angle exists
  projected := normalize(p1·ort1 + p2·ort2)
  angle := arccos(ort1 · projected)
  IF ort2 · projected < 0 THEN angle := -angle
  RETURN angle
```

**Notes** — the projection back into the plane is what makes this robust. Without it, a rotation
whose axis differs from the queried axis gives a turned vector that has left the plane, and the
arc-cosine of its dot product is not an angle about the queried axis at all — it is smaller, and the
error grows with the misalignment. Joints are queried about axes that are only approximately the
true rotation axis all the time, which is why the correction matters in practice.

The answer is confined to the half-open turn from minus half a turn to half a turn. Stops outside
that range are folded by the limit code, not here; see `CalcAxis` in
[`PHJoint.cpp`](PHJoint.cpp.md).

## Exported units

**Construction and lifecycle** — construct(type, first, second), `Create`, `RunSimulation`,
`Activate`, `Deactivate`, `SetShell`, `SetBackRef`, `ReattachFirstElement`.

**Anchor** — `SetAnchor`, `SetAnchorVsFirstElement`, `SetAnchorVsSecondElement`,
`GetAnchorDynamic`.

**Axes** — `SetAxisDir` and its two frame-relative forms, `SetAxisDirDynamic`, `GetAxisDir`,
`GetAxisDirDynamic`, `GetAxesNumber`, `SetAxis`.

**Stops** — `SetLimits`, `SetLimitsVsFirstElement` / `SetLimitsVsSecondElement` (both empty; see
the implementation twin), `SetHiLimitDynamic`, `SetLoLimitDynamic`, `GetLimits`.

**Motors** — `SetForce`, `SetVelocity`, `SetForceAndVelocity`, `GetMaxForceAndVelocity`.

**Softness** — `SetAxisSDfactors`, `SetJointSDfactors`, `GetAxisSDfactors`, `GetJointSDfactors`,
`SetJointFudgefactorActive`, and the three `…Active` appliers.

**Readout** — `GetAxisAngle`, `GetAxisAngleRate`, `IsWheelJoint`, `IsHingeJoint`, `BoneID`,
`PFirst_element`, `PSecond_element`.

**Breakability** — `SetBreakable`, `isBreakable`, `JointDestroyInfo`, `ClearDestroyInfo`,
`RootGeom`.
