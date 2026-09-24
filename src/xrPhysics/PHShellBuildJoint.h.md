# src/xrPhysics/PHShellBuildJoint.h

> Reads a bone's three authored angular limits and decides, from which of them are degenerate, whether the joint is a hinge, a three-axis limited rotation, a wheel, a slider or a free ball.

**Needs** — [`PHJoint.h`](PHJoint.h.md) · [`PHElement.h`](PHElement.h.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHShell.cpp`](PHShell.cpp.md)
**Tier floor** — T2: it is a decision table over authored numbers.

## Purpose

The bridge from the model format's joint description to a constructed constraint. It is a header with
no implementation because it is used from exactly one place — the shell's build walk in
[`PHShell.cpp`](PHShell.cpp.md) — and a rebuild should simply make it a function there.

The load-bearing content is small and easy to miss: **a model does not say "this is a hinge".** It
says "this bone has three angular limits", and the joint kind is *inferred* from which of those
limits are degenerate. That inference is the decision a rebuild must reproduce, because it is what
every shipped model's joints depend on.

## the authored joint description

```text
RECORD BoneJointData                  # what the model file carries per bone
  type            : one of {none, rigid, cloth, joint, wheel, slider}
  limits          : 3 × (low, high, spring_factor, damping_factor)
  spring_factor   : real              # the JOINT's own softness, distinct from the limits'
  damping_factor  : real
  friction        : real              # becomes the joint's motor force
  breakable       : bool
  break_force, break_torque : real
```

**Invariants** — a limit whose low and high are *equal* means "this axis does not move". A limit
spanning a full turn or more means "this axis is free". Everything between is a real stop. Those two
degeneracies are the entire type inference.

## `BuildJoint` — the dispatch

```text
FUNCTION build_joint(bone, parent_element, element) -> optional<Joint>
  joint := CASE bone.joint.type OF
    slider → build_slider(bone, …)
    cloth  → build_ball(bone, …)          # a free ball-and-socket
    joint  → build_generic(bone, …)       # infer from the limits
    wheel  → build_wheel(bone, …)
    none   → none
  IF joint THEN joint.set_force_and_velocity(bone.joint.friction)
  RETURN joint
```

**Notes** — the authored `friction` becomes the joint's **motor force with a target velocity of
zero**, which is exactly a friction model: the joint resists motion with up to that much force and
is otherwise free. Nothing else in the engine implements joint friction, and this one line is it.
A rebuild whose solver has a real joint-friction term should use that instead and will get the same
behaviour more cheaply.

A `rigid` joint never reaches here: the build walk fuses such a bone into its parent's body and
builds no joint at all. `none` produces no joint, leaving two bodies in one shell with nothing
holding them together — legal, and used for objects whose parts are meant to be separate from the
start.

## `BuildGenericJoint` — the inference

**Contract** — chooses between a hinge and a three-axis limited rotation by counting how many of the
three limits are degenerate, and — this is the part that must be reproduced exactly — *which axes
are assigned to which limits* in each case.

```text
FUNCTION build_generic(bone, …) -> Joint
  eq[i] := (limits[i].low is indistinguishable from limits[i].high)   for i in 0,1,2

  IF eq[0] AND eq[1] THEN hinge about the THIRD coordinate direction, using limit 2
  IF eq[0] AND eq[2] THEN hinge about the SECOND, using limit 1
  IF eq[1] AND eq[2] THEN hinge about the FIRST,  using limit 0

  # exactly one frozen axis, or none at all: a three-axis limited rotation,
  # with the axis order chosen so the FROZEN one is not the middle axis
  IF eq[0] THEN full_control with limit order (2, 0, 1)
  IF eq[1] THEN full_control with limit order (2, 1, 0)
  IF eq[2] THEN full_control with limit order (0, 2, 1)
  OTHERWISE  full_control with limit order (2, 0, 1)
```

**Invariants** — two frozen axes give a hinge about the third. Fewer give a three-axis joint. The
axis *orderings* are not interchangeable and this is the subtle constraint: a three-axis angular
limit decomposes a rotation into a sequence, and the middle axis of that sequence is the one that
loses a degree of freedom when the outer two align — the familiar gimbal degeneracy. The orderings
are chosen so that the *most constrained* limit sits at the middle, where the degeneracy costs
least. A rebuild that picks a different order will find ragdoll limbs flipping at particular poses.

The fall-through when *no* limit is degenerate uses the same order as the first-frozen case, which
is a default rather than a derivation.

**Notes** — only two of the three axes are given explicit directions; the middle one is left to the
solver, which derives it as perpendicular to the other two. That is inherent to the three-axis
decomposition — specifying all three would over-determine it — and a rebuild must supply only the
outer two.

## `CtreateHinge`

**Contract** — a one-axis joint about a named coordinate direction of the *child* body, with that
limit's stops and its own spring and damping.

## `CtreateFullControl`

**Contract** — a ball-and-socket for position plus a three-axis angular limit, with the first and
third axes named against the child body and all three limits installed.

## `BuildWheelJoint`

**Contract** — a hinge-2: the suspension and steering axis is the child body's first coordinate
direction, the spin axis its third. Limit 0 becomes the steering stop; the spin axis is left free.

**Notes** — this is the vehicle joint and the axis convention is a hard contract with the model
data: every wheel bone in every shipped vehicle is authored expecting *first direction steers,
third direction spins*. The suspension itself is the hinge-2's own sprung first axis, tuned by the
joint-level spring and damping factors — which is why the hinge-2 has its own much stiffer defaults
in [`PHJoint.cpp`](PHJoint.cpp.md). A rebuild must map both axes the same way or every car in the
game drives sideways.

## `BuildSliderJoint`

**Contract** — a translation along an axis plus one rotation about another. Limit 0 gives the
translation's range — installed **directly, without the free-limit test** — and limit 1 the
rotation's, installed through it.

**Notes** — the asymmetry is correct and worth stating: a *translation* limit spanning a full turn is
not degenerate, it is a distance, so the free-axis test (which asks whether the span reaches a full
turn) is meaningless for it. Applying it would make a slider with a long travel silently unbounded.

## `IsFreeRLimit` and `SetJointRLimit`

**Contract** — a rotational limit is *free* when its span reaches a full turn; a free limit installs
no stops at all, leaving the axis unbounded.

**Notes** — the distinction matters because installing stops that happen to span a full turn is not
the same as installing none: the solver still evaluates them, still folds them into its own range
(see `CalcAxis` in [`PHJoint.cpp`](PHJoint.cpp.md)), and can push against them near the fold. A
genuinely free axis must have no stops.

## `SetJoint`

**Contract** — the two things every joint gets regardless of kind: its anchor at the **child body's
origin**, and the bone's joint-level spring and damping.

**Invariants** — the anchor is always the child's origin, in the child's frame. That is a hard
convention of the model format — a bone's origin *is* its joint — and it is why no joint in the
engine ever needs an authored anchor position.
