# src/xrGame/ik — foot placement by inverse kinematics

Part of chapter 26 of [`SYSTEM-REQUIREMENTS.md`](../../../SYSTEM-REQUIREMENTS.md#7-build-order),
alongside [`CdkeyDecode`](../CdkeyDecode/README.md) and [`gamespy`](../gamespy/README.md),
with which it shares nothing but a chapter number.

Every walk cycle in the game was authored on a flat floor. The world is not flat. Played
back unaltered on a slope, a stair tread or a rock, a creature's feet hang in the air or
sink through the surface, and the error is large enough to see — a few centimetres is a
visible float, a step's worth is a visible clip. This directory fixes that: once per
frame, after the animation has posed the skeleton and before the pose is used, each leg's
foot is moved to meet the surface actually under it, the leg is re-solved to reach the
moved foot, and the whole body is slid vertically so that no leg is asked to reach further
than it can.

The correction is deliberately **local and late**. Nothing upstream knows it happened: the
animation system produced the same pose it always would, the physics capsule is unmoved,
the navigation position is unchanged. Only the bone transforms handed to the renderer
differ. This is what makes the feature optional — a rebuild that skips this directory
produces a game that plays identically and looks slightly wrong on slopes.

## Where it sits

It is the last thing that touches a skeleton in a frame, and it therefore rests on almost
everything: the skeleton and its animation blends, the static and dynamic collision
database (through the level's ray-pick service), the surface material table (to decide
which surfaces a foot may stand on), and the frame clock. Nothing depends on it.

The directory is **two layers that a rebuilder should keep separate**:

- **A general analytic 7-degree-of-freedom limb solver.** It is the IKAN library from the
  University of Pennsylvania (2000), vendored with the engine's own float and math types
  substituted in. It knows nothing about feet, ground or animation. Files:
  [`limb`](limb.cxx.md), [`Dof7control`](Dof7control.cpp.md), [`eulersolver`](eulersolver.cxx.md),
  [`jtlimits`](jtlimits.cxx.md), [`eqn`](eqn.cxx.md), [`aint`](aint.cxx.md),
  [`math3d`](math3d.cpp.md), [`mathTrig`](mathTrig.cpp.md).
- **The engine's use of it**: [`IKLimb`](IKLimb.cpp.md), which owns one leg, asks the
  world where the ground is, decides what the foot's goal should be, rate-limits the
  change, and drives the solver.

The per-frame orchestration, the foot geometry, the ground query itself and the
whole-body vertical shift live one directory up, in `src/xrGame` — see
[`IKLimbsController.cpp`](../IKLimbsController.cpp.md), [`IKFoot.cpp`](../IKFoot.cpp.md),
[`ik_foot_collider.cpp`](../ik_foot_collider.cpp.md) and
[`ik_object_shift.cpp`](../ik_object_shift.cpp.md). That split is historical (this
directory is the vendored library plus the one file that adapts it) and a rebuilder is
free to ignore it.

## Load-bearing ideas, named once

**The chain is four bones and seven degrees of freedom.** Hip (3 rotational degrees),
knee (1), ankle (3), toe (0 — it rides along, but it is a named bone because it is one of
the two candidate contact references). Named by convention as
`bip01_l_thigh, bip01_l_calf, bip01_l_foot, bip01_l_toe0` and the right-side and arm
equivalents, overridable per visual through the model's own configuration. Seven degrees
reaching a six-degree goal leaves exactly **one** free parameter, and naming that
parameter is the whole trick of the solver.

**That free parameter is the swivel angle.** Fix the hip and the foot; the knee is then
constrained to a circle whose plane is perpendicular to the hip→foot line. The position of
the knee on that circle is the swivel angle ψ. Every other joint value follows from the
goal and ψ in closed form — no iteration, no Jacobian, no convergence criterion. This is
why the solve costs a fixed handful of microseconds and can run on every leg of every
visible creature every frame.

**The chain is solved in the hip's frame, and the goal is a full matrix.** The foot's goal
carries orientation as well as position, because a foot that reaches the right point with
the wrong tilt is exactly as wrong as one that misses. The chain equation the solver
inverts is `G = R2 · S · Ry · T · R1`, where `T` and `S` are the fixed bind-pose
hip→knee and knee→ankle transforms, `Ry` is the single knee rotation, and `R1`, `R2` are
the hip and ankle rotations to be found.

**The knee angle is a law of cosines.** Two fixed link lengths and a required hip→foot
distance determine the knee's bend regardless of ψ, so it is solved first and separately.
This is also where reachability is enforced: a goal further away than the leg is long is
pulled back to 0.9999 of the leg's length before anything else runs, so the solver is
never handed an impossible problem and never has to report failure mid-frame.

**The swivel is inherited from the animation, not chosen freely.** The solver can pick a ψ
that best satisfies joint limits, but the engine does not let it: it takes the knee
position the animator posed, re-expresses it relative to the *new* hip→foot direction, and
converts that to a ψ. The rule is that IK corrects where the foot is, and nothing else —
the knee keeps pointing the way the animator pointed it. A rebuild that lets the solver
choose ψ will get legs that twist visibly as the ground changes.

**Joint limits are computed analytically and then switched off.** The library can, for a
given goal, compute the exact *set of angles* ψ for which every joint stays inside its
limits, as a list of intervals on the circle — that is what
[`jtlimits`](jtlimits.cxx.md), [`eqn`](eqn.cxx.md) and [`aint`](aint.cxx.md) exist for,
and it is genuinely elegant. The shipping engine passes "limits off" on every call and
never evaluates any of it; limits are enabled only behind a debug draw flag. A rebuild
should implement the limit machinery last, or not at all, and rely instead on the fact
that the goal is a small perturbation of a pose an animator already made legal.

**Two candidate contact references: the foot bone and the toe bone.** Which one the goal
is expressed about changes during a step — a heel-strike is referenced to the foot, a
toe-off to the toe — and the choice is re-made each frame from the pose. Everything
downstream carries a "reference bone" alongside the goal matrix for this reason.

**The ground is queried with three rays, not one.** A foot is a polygon, not a point, so
the surface under it is sampled at toe, heel and one side point. If all three hits are
close enough together to be the same surface (within 1.5 foot lengths), a plane is fitted
through the three hit points and the foot is aligned to *that* — which is what makes a
foot lie flat across a slope rather than pivot on its toe. If they disagree, only the toe's
own triangle plane is used. Rays start half a metre *above* the sample point and run two
metres, so a foot already slightly inside geometry still finds the surface it is standing
on. Surfaces marked passable or climbable, and living creatures, are transparent to the
query — the ray restarts past them rather than stopping.

**Correction is rate-limited, not weighted.** There is no blend factor. Each frame the new
goal is clamped to a maximum linear and angular *change* from last frame's goal; the
allowance starts at the speed the animation itself is moving the foot (×1.5) and
accelerates while a correction is outstanding. When the clamp stops binding, blending is
over. The consequence a rebuilder must reproduce: a foot never snaps, and a foot that has
been standing still for a moment tracks the ground exactly.

**A planted foot is frozen, with an escape hatch.** While the animation's own footstep
marks say the foot is on the ground, the goal is pinned to the contact pose chosen when it
was planted, so the foot does not slide under a moving body. An idling creature re-picks
that contact pose if the ground under it has moved more than 0.3 m or 45°, or after a
randomized 0.5–1.2 s if it has moved at all — the randomization exists so that several
creatures standing together do not re-plant in lockstep.

**The hip is adjusted by moving the entire body.** Nothing rotates the pelvis. Instead one
scalar vertical offset is applied to every bone of the skeleton after the pose is built.
Its target is the compromise between two demands: planted feet that had to be raised want
the body raised by their average, and every planted leg has a maximum depth below which it
would have to stretch past its own length — that limit always wins. The offset is driven
toward its target by a smooth position/velocity/acceleration/jerk polynomial rather than
snapped, and it is capped at one metre.

**The body is lowered *before* the foot lands.** For a foot currently in the air, the
engine reads the animation forward to find when the next footstep mark fires,
extrapolates where the body will be at that moment, runs the same ground query against the
*predicted* foot pose, and starts moving the vertical offset now so it arrives on time.
This is what makes a creature descending stairs look like it is stepping down rather than
being pushed down.

**The solver's matrix convention is not the engine's.** The vendored library is
row-vector, Y-up-with-a-different-handedness; the engine is not. Every matrix crossing the
boundary is conjugated by one fixed permutation. This is pure impedance matching and a
rebuild that writes the solver itself should delete it.

## The twins

| File | Role |
|---|---|
| [`IKLimb.h`](IKLimb.h.md) | Surface of the engine-side limb. |
| [`IKLimb.cpp`](IKLimb.cpp.md) | One leg: goal selection, ground fitting, rate-limited blending, plant/unstuck, driving the solver, writing bone transforms. The substantial page of the directory. |
| [`limb.h`](limb.h.md) | Surface of the general 7-DOF limb. |
| [`limb.cxx`](limb.cxx.md) | The limb as a *joint-space* problem: Euler extraction, family selection, swivel choice under limits. |
| [`Dof7control.h`](Dof7control.h.md) | Surface of the S-R-S position/orientation solver. |
| [`Dof7control.cpp`](Dof7control.cpp.md) | The closed-form geometry: knee angle, swivel circle, recovery of the two spherical rotations. |
| [`eulersolver.h`](eulersolver.h.md) | Surface of Euler decomposition and its swivel-parameterized form. |
| [`eulersolver.cxx`](eulersolver.cxx.md) | Rotation matrix → three joint angles under a named convention, both solution families, and the same as a function of ψ. |
| [`jtlimits.h`](jtlimits.h.md) | Surface of the per-joint feasible-ψ machinery. |
| [`jtlimits.cxx`](jtlimits.cxx.md) | Turning one joint's angular limits into the set of ψ that satisfy them. |
| [`eqn.h`](eqn.h.md) | Surface of the sinusoid-in-ψ primitive. |
| [`eqn.cxx`](eqn.cxx.md) | `α·cos ψ + β·sin ψ + ξ`: its roots, critical points and level sets. Everything above is built from this one shape. |
| [`aint.h`](aint.h.md) | Surface of angular intervals and sets of them. |
| [`aint.cxx`](aint.cxx.md) | Intervals on a circle: containment, distance, union, intersection, merging. Wrap-around is the whole difficulty. |
| [`math3d.h`](math3d.h.md) | Surface of the vendored linear algebra. |
| [`math3d.cpp`](math3d.cpp.md) | The library's own vectors, 4×4 transforms, quaternions and rotation constructions — parallel to the engine's, and replaceable by them. |
| [`mathTrig.h`](mathTrig.h.md) | Surface of the trigonometric solvers, plus the law of cosines. |
| [`mathTrig.cpp`](mathTrig.cpp.md) | Closed-form solutions of the two trigonometric equations the geometry reduces to. |
