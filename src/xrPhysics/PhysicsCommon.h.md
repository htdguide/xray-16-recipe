# src/xrPhysics/PhysicsCommon.h

> The rate conversion between authored stiffness and the solver's soft-constraint parameters, plus the module's global tuning constants.

**Needs** — [`DisablingParams.h`](DisablingParams.h.md) · [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md) · [`Physics.cpp`](Physics.cpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`Geometry.h`](Geometry.h.md) · [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md) · [`PHActivationShape.cpp`](PHActivationShape.cpp.md) · [`PHActorCharacter.cpp`](PHActorCharacter.cpp.md) · [`PHCapture.cpp`](PHCapture.cpp.md) · [`PHCaptureInit.cpp`](PHCaptureInit.cpp.md) · [`PHContactBodyEffector.cpp`](PHContactBodyEffector.cpp.md) · [`PHElement.h`](PHElement.h.md) · [`PHInterpolation.cpp`](PHInterpolation.cpp.md) · [`PHIsland.cpp`](PHIsland.cpp.md) · [`PHIsland.h`](PHIsland.h.md) · [`PHJoint.cpp`](PHJoint.cpp.md) · [`PHJointDestroyInfo.cpp`](PHJointDestroyInfo.cpp.md) · _and 6 more_
**Tier floor** — T2: arithmetic and shared constants. Defined in [`Physics.cpp`](Physics.cpp.md).

## Purpose

A soft constraint in the dynamics library is described by two numbers — an *error reduction*
fraction and a *constraint force mixing* term — and both depend on the timestep. Authored data
(material stiffness, joint spring and damping factors in the model files) describes a spring and a
damper, which do not. This header is the conversion between the two representations, and it is used
at every single site that softens a contact or a joint limit.

Getting this wrong is not subtle: every surface in the game becomes uniformly harder or softer, and
jointed structures either sag or buzz.

## State

The declared globals, with their role:

```text
default_l_limit   = 150      # linear velocity ceiling for a rigid body, metres/second
default_w_limit   ≈ 9.817    # angular ceiling, radians/second — a sixteenth-turn per base step
default_l_scale   = 1.01     # per-step linear velocity divisor: a permanent ~1% bleed
default_w_scale   = 1.01     # the same for angular velocity
default_k_l       = 0.0002   # linear air resistance; force is proportional to speed *squared*
default_k_w       = 0.05     # angular air resistance; torque proportional to angular speed

base_fixed_step   = 0.02     # the rate the base stiffness pair was authored at
base_erp, base_cfm           # the authored stiffness, expressed as the library's pair at that rate

fixed_step        = 0.01     # the rate actually used; overridable from configuration
world_cfm, world_erp         # the same stiffness re-expressed at fixed_step
world_spring, world_damping  # ... and as a rate-independent spring/damper pair

max_joint_allowed_for_exact_integration = 30   # island size above which the exact solver is refused
default_world_gravity = 2 × 9.81               # twice Earth gravity
gravity_time_factor, solver_iteration_count    # script- and console-settable
```

**Invariants** — `default_w_limit` is exactly a π/16 turn per `base_fixed_step`, and
`default_l_limit` is exactly 3 metres per `base_fixed_step`. Both are *displacement per step*
budgets rescaled to per-second: the real constraint is that a body must not move more than a
fraction of its own size in one step, or the collider will miss the contact. A rebuild changing the
step should recompute these from the displacement budget, not copy the numbers.

**Notes** — gravity is **twice** the real value. This is a game-feel decision that runs through
every mass and joint value in the shipped data: objects fall fast and settle hard. Halving it to
be "correct" invalidates the entire authored tuning set, which is exactly what
[§3's rigid-body seam note](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) warns about.

The `1.01` scales are a curiosity: every step, a body's velocity is *divided* by them, which is a
uniform 1%-per-step energy drain unrelated to air resistance or friction. It exists to stop the
iterative solver's error from accumulating into perpetual motion in a resting stack. A rebuild with
a solver that does not gain energy can drop it; one that keeps it must apply it at the same point
in the step (post-solve, pre-damping — see [`PHElement.cpp`](PHElement.cpp.md)) or the numbers
mean something else.

## The conversion

**Contract** — four pure functions and their macro equivalents. Spring and damping are the
authored, rate-independent form; error-reduction and force-mixing are the solver's form.

```text
FUNCTION erp(spring, damping, step)  -> step·spring / (step·spring + damping)
FUNCTION cfm(spring, damping, step)  -> 1 / (step·spring + damping)
FUNCTION spring_of(cfm, erp, step)   -> erp / cfm / step
FUNCTION damping_of(cfm, erp)        -> (1 - erp) / cfm
```

These are exact inverses; the pairs `(spring, damping)` and `(erp, cfm)` carry the same
information at a fixed step.

## `mul_spring_damping`

**Contract** — scales an existing `(cfm, erp)` pair by independent spring and damping multipliers
*without* a round trip through the spring/damper form. Used on the hot path where a contact's
softness is adjusted per material or per character state.

```text
FUNCTION mul_spring_damping(INOUT cfm, INOUT erp, spring_mul, damping_mul)
  factor := 1 / (spring_mul·erp + damping_mul·(1 - erp))
  cfm := cfm · factor
  erp := erp · factor · spring_mul
```

**Notes** — this is the algebraic simplification of "convert to spring/damper, multiply each, and
convert back". Doing it in one step avoids two divisions per contact, and a rebuild may equally
well write the round trip if the contact count is low enough. The identity is worth checking on
arrival: with both multipliers 1, the pair must be unchanged.
