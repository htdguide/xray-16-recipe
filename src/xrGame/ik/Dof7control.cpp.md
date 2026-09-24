# src/xrGame/ik/Dof7control.cpp

> The geometry. Two link lengths and a required reach fix the knee's bend by the law of
> cosines; the knee is then free to swing on a circle whose plane is perpendicular to the
> hip-to-foot line; and once you name a point on that circle, both ball joints fall out as
> the rotation carrying one pair of vectors onto another. No iteration anywhere.

**Needs** — [`Dof7control.h`](Dof7control.h.md) · [`math3d.h`](math3d.h.md) · [`mathTrig.h`](mathTrig.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2. A few dozen multiplies and four transcendental calls per solve, on
fixed-size data, with no allocation. Nothing needs explicit layout.

## Purpose

This is the load-bearing page of the vendored solver and the one a rebuilder should read
first. It answers: given `G = R2 · S · Ry · T · R1`, find `R1`, `Ry`, `R2`. The answer is
in closed form and is reached in three separate steps, each of which is independently
worth naming because each has its own failure mode:

1. **The hinge.** Depends only on the goal's *distance* — not its direction, not its
   orientation, and not the swivel angle. Solved first, once.
2. **The circle.** With the hinge fixed, the hinge joint's position is constrained to a
   circle. Its centre, radius and plane are computed once per goal.
3. **The two ball joints.** Given any point on that circle, both follow as frame-to-frame
   rotations. This is the only step that depends on the free parameter.

## State

```text
RECORD ChainSolver
  T, S            : Transform     # fixed: root->hinge, hinge->tip. Read from the bind pose.
  upper_len       : real          # |translation of T|
  lower_len       : real          # |translation of S|
  proj_axis       : vector        # where the swivel angle is measured from
  pos_axis        : vector        # which way round the swivel angle counts
  project_to_workspace : bool     # clamp an unreachable goal instead of failing

  # per-goal, valid only after a goal is posed
  G               : Transform     # the goal, possibly with its position clamped
  ee              : vector        # goal position, in the root's frame
  hinge_angle     : real
  SRT             : Transform     # S * Ry * T, the whole chain minus the ball joints
  ee_r1           : vector        # where the tip sits before R1 is applied
  p_r1            : vector        # where the hinge sits before R1 is applied
  c, u, v, n      : vector        # the swivel circle: centre, in-plane axes, normal
  radius          : real
```

**Invariants** —

- `T` and `S` are rigid and never change during a solve. Their translations' lengths are
  the two link lengths and are cached at initialization; a rebuild that lets a link length
  change must recompute them.
- After any goal is posed, `ee` is **guaranteed reachable**. That is the entire job of the
  workspace clamp below, and every step after it assumes it.
- `n`, the circle's normal, is the unit hip-to-foot direction — *possibly negated*. See
  the note on `pos_axis`.
- The swivel angle is meaningful only relative to the `(u, v, n)` frame, which depends on
  the goal. The same numeric angle means a different knee position for a different goal.
  Carrying a swivel angle across frames — which the engine does — is therefore only valid
  because the goal changes slowly.

## The workspace clamp

**Contract** — before anything else, if the goal is further from the root than the two
links together, it is pulled back along its own direction to **0.9999** of the chain's
reach. Reports whether it moved. The shorter-than-reachable case (a goal inside the inner
workspace boundary, which a chain with unequal links also cannot reach) is present in the
source and disabled.

```text
FUNCTION clamp_into_reach(upper_len, lower_len, goal) -> bool
  max_len <- (upper_len + lower_len) * 0.9999
  IF |goal| <= max_len RETURN false
  goal <- goal scaled to length max_len
  RETURN true
```

**Notes** — this is the single most important decision in the file and it is three lines.
**The solver is never allowed to be handed an impossible problem**, so it never has to
report failure in the middle of a frame, and the caller never has to decide what to do
about a leg that could not reach. A foot placed too far away simply arrives at a
fully-extended leg pointing at it.

The factor is 0.9999 rather than 1.0 because at exactly full extension the law of cosines
gives a hinge angle of zero, the swivel circle collapses to a point of zero radius, and
every subsequent frame construction divides by that radius. One part in ten thousand of a
metre-long leg is a tenth of a millimetre — invisible, and enough to keep the circle
non-degenerate.

Disabling the clamp is exposed as a switch and this engine never uses it. A rebuild may
drop the switch.

## `SetGoal`

**Contract** — pose a full position-and-orientation goal. Stores it (with its position
clamped into reach), computes the swivel circle, solves the hinge angle, and caches the
composite `S · Ry · T`. Returns the hinge angle and whether a solution exists — the only
failure is the law of cosines failing, which the clamp is supposed to make impossible.
Allocates nothing.

```text
FUNCTION set_goal(solver, G) -> optional<real>
  solver.G  <- G
  ee        <- translation of G
  p_r1      <- translation of T            # hinge position before the root rotation
  s         <- translation of S

  IF project_to_workspace AND clamp_into_reach(|p_r1|, |s|, ee)
    write the clamped position back into solver.G

  evaluate_circle(ee)
  hinge_angle <- solve_hinge(ee, s, p_r1, T)  OR RETURN none
  hinge_angle <- -hinge_angle                 # sign convention; see note

  # Everything from the hinge outward is now fixed, independent of the swivel angle.
  SRT   <- S · rotation_about_y(-hinge_angle) · T
  ee_r1 <- translation of SRT                 # where the tip sits before R1
  RETURN hinge_angle
```

**Notes** — the hinge angle is negated immediately after it is solved and then negated
again when the rotation is built, so the two cancel. That is the impedance match between
the solver's hinge sense and the skeleton's; it survives as *the hinge's positive
direction is a convention you must fix once and state*.

The position-only variant differs in three ways: the "lower link" length is the tip
transform extended by a caller-supplied constant transform (so the point being aimed is
not the ankle but something rigidly attached to it), the hinge angle is not negated, and
no goal orientation is stored — so only the root ball joint can afterwards be solved.

## The hinge angle

**Contract** — given the goal position and the two link vectors, the hinge rotation that
makes the chain's tip land at the goal's *distance*. Returns zero, one or two solutions
and keeps one; failure means no triangle closes.

The derivation, which is worth carrying because it explains why the answer does not depend
on the swivel angle:

```text
# The distance from root to tip must equal the distance to the goal:
#     (s · Ry · T) · (s · Ry · T)ᵀ  =  g · gᵀ
# Expanding and cancelling the terms that do not involve Ry leaves
#     s · Rot(Ry) · Rot(T) · tᵀ  =  g·gᵀ - s·sᵀ - t·tᵀ
# which, because Ry is a rotation about one axis, is
#     a·cos(theta) + b·sin(theta) = c
# with a, b built from the components of Rot(T)·tᵀ against s, and c the right-hand
# side above. Solved by solve_trig1.

FUNCTION solve_hinge(g, s, t, T) -> optional<real>
  rhs   <- dot(g,g) - dot(s,s) - dot(t,t)
  alpha <- Rot(T) applied to t            # note: Rot(T)·tᵀ, NOT t·Rot(T)
  a <- 2 * (alpha.x*s.x + alpha.z*s.z)
  b <- 2 * (alpha.x*s.z - alpha.z*s.x)
  c <- rhs - 2 * (alpha.y*s.y)
  answers <- solve_trig1(a, b, c)
  IF answers is empty RETURN none
  # Two answers are the knee bent one way and the other. Prefer a negative one —
  # for a leg that is the anatomically possible direction.
  RETURN the negative answer if one exists, else the second answer
```

**Notes** — only the *y* components drop out of `a` and `b` because the hinge rotates
about *y*; that is where the convention "the hinge axis is *y* in the solver's frame"
becomes load-bearing rather than cosmetic.

The selection among two answers reads as unfinished: when both are negative it takes the
first, when both are positive it takes the second with a question mark left in the source.
**Why the second rather than the first in the both-positive case is not recoverable.** In
practice a leg goal always yields one negative answer, so the ambiguous branches are not
reached. A rebuild should choose by the joint's authored sign convention instead of by
these cases.

The whole function can be replaced by the law of cosines on the two link lengths and the
goal distance, which is what [`limb.cxx`](limb.cxx.md) does when it only needs the angle.
The longer form survives because it also works when the hinge axis is not perpendicular to
both links — a chain whose bones are not coplanar in the bind pose.

## The swivel circle

**Contract** — given the tip position, the centre, radius, plane normal and two in-plane
axes of the circle on which the hinge joint may lie. Returns a radius of zero when the law
of cosines fails, which after the clamp does not happen.

```text
FUNCTION evaluate_circle(ee, proj_axis, pos_axis, upper_len, lower_len)
  reach <- |ee|
  n     <- ee normalized                  # the circle's plane is perpendicular to this

  # The angle at the root between the upper link and the root-to-tip line. Fixed by
  # the three lengths alone — this is where the hinge's bend becomes the circle's size.
  alpha <- law_of_cosines(reach, upper_len, lower_len)  OR RETURN 0

  c      <- n scaled by cos(alpha) * upper_len          # centre, along the reach line
  radius <- sin(alpha) * upper_len

  # If the goal lies behind the root rather than in front of it, flip the normal so
  # that the swivel angle keeps counting the same way round. Without this the angle
  # reverses sign the moment a foot crosses behind the hip, and a knee snaps.
  IF dot(n, pos_axis) < 0
    n <- -n

  u <- proj_axis with its component along n removed, normalized   # the zero of psi
  v <- n × u                                                      # the quarter turn
```

**Notes** — this is the geometric heart. Three facts follow from it and a rebuilder should
hold all three:

- **The circle's size depends only on distances**, so it is fixed the moment the hinge
  angle is. The free parameter genuinely is free.
- **`u` is where the swivel angle reads zero**, and it is the projection axis with the
  reach direction removed. Change the projection axis and every swivel angle in the system
  shifts by a constant. The engine fixes it once at limb setup.
- **The normal flip is not cosmetic.** It is the difference between a swivel angle that is
  continuous as a foot swings from in front of the hip to behind it and one that jumps a
  half turn. It costs one dot product and it is the kind of thing a rebuild omits and then
  cannot explain.

## `PosToAngle` · `AngleToPos`

**Contract** — the free parameter, in both directions. A hinge position becomes a swivel
angle by projecting it into the circle's plane and measuring the signed angle from `u`
about `n`. A swivel angle becomes a position by `c + radius·cos ψ·u + radius·sin ψ·v`.
Neither validates that the position is on the circle — a position off the circle projects
onto it silently, which is exactly what the caller wants: the engine hands in the knee
position the animator posed, which is *not* on the new circle, and gets back the nearest
swivel angle to it.

**Notes** — that tolerance of off-circle input is the mechanism behind the rule stated in
the chapter README: *the swivel is inherited from the animation, not chosen freely.* A
rebuild that asserts the input lies on the circle breaks the feature.

## `SolveR1` · `SolveR1R2`

**Contract** — given a swivel angle (or a hinge position, which is converted first),
recover the root ball joint, and optionally the tip ball joint as well. No failure case.

```text
FUNCTION solve_root(psi) -> Transform
  p <- point on the circle at psi
  # Two frames built from the same pair of points, once before the root rotation and
  # once after. The rotation between them IS the root ball joint.
  #   before: hinge at p_r1, tip at ee_r1
  #   after:  hinge at p,    tip at ee
  RETURN frame(p_r1, ee_r1)ᵀ · frame(p, ee)

FUNCTION solve_both(psi) -> (R1, R2)
  R1 <- solve_root(psi)
  # Everything except R2 is now known, so R2 is what remains of the goal.
  R2 <- G · (SRT · R1)⁻¹
```

where `frame(p, q)` builds an orthonormal basis with its first axis along `p`, its second
along the part of `q` perpendicular to `p`, and its third their cross product.

**Notes** — three points on a rigid body determine its orientation, and here the third is
the root itself, at the origin of both frames. So two points suffice and the construction
is a Gram-Schmidt on them. The first axis is normalized by *multiplying* by a reciprocal
length cached at initialization — the upper link's length, which is what `|p|` and
`|p_r1|` both equal by construction. That is a real invariant, not an optimization detail:
if the circle's radius and centre were computed correctly, the hinge is exactly one upper
link from the root, and the code relies on it.

The tip joint is pure subtraction — it is whatever rotation the goal still demands once
the rest of the chain is placed. It cannot fail and it cannot be constrained, which is why
the ankle's joint limits can only ever be *checked*, never *satisfied by construction*.

## `R1Psi` · `R1R2Psi`

**Contract** — express the ball joints not as matrices but as **functions of the swivel
angle**: three matrices `C`, `S`, `O` such that `R1(ψ) = cos ψ·C + sin ψ·S + O`, and
optionally another three for `R2`. This is what makes the closed-form joint-limit analysis
in [`jtlimits.cxx`](jtlimits.cxx.md) possible at all; it is otherwise unused.

```text
FUNCTION root_as_function_of_psi() -> (C, S, O)
  R0 <- solve_root(0)                    # the root joint at psi = 0
  # Rotating the knee around its circle by psi is a rotation about the circle's
  # normal by psi, applied after R0. Rodrigues' formula decomposes any such rotation
  # into exactly a cosine part, a sine part and a constant part:
  #   R(n, psi) = cos(psi)*(I - nnᵀ) + sin(psi)*[n]× + nnᵀ
  (C, S, O) <- that decomposition for the normal n
  RETURN (R0·C, R0·S, R0·O)

FUNCTION both_as_functions_of_psi() -> (C, S, O, C2, S2, O2)
  (C, S, O) <- root_as_function_of_psi()
  # R2 = G · (SRT · R1)⁻¹ is not linear in R1, so this is NOT a decomposition of R2
  # in the same sense — each of the three parts is transformed separately and the
  # result is a decomposition only because the downstream analysis reads entries,
  # not matrices.
  FOR EACH part IN (C, S, O)
    emit G · (SRT · part)⁻¹
```

**Notes** — Rodrigues' decomposition into three constant matrices is the trick that turns
"a joint angle as a function of ψ" into "a sinusoid in ψ with known coefficients", and it
is the single reason the joint-limit analysis is exact rather than sampled. Worth
implementing even if the limit machinery is not, because it is four lines and it documents
the structure.

## `SetAimGoal` · `SolveAim`

**Contract** — a different problem posed on the same chain: point a named axis of the tip
at a target, with the hinge angle *supplied* rather than solved, and with orientation
about the aiming axis left free. Solves only the root joint. The circle here is the locus
of tip positions, not hinge positions — a triangle is closed between the root, the tip and
the target, and the tip's distance from the root is derived from the hinge angle by
composing the two links through it.

**Notes** — this is the arm case. The shipped game configures four limbs and enables only
the two legs; the arm bone names are present and no creature turns them on. A rebuild
should implement this last, and the comment in the source calling for the whole thing to
be rewritten should be taken at face value: it duplicates the circle construction with a
second, differently-derived version, and the two do not share code.
