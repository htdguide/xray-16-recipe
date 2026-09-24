# src/xrGame/ik/limb.cxx

> Between the geometry and the skeleton sits one decision the geometry cannot make: of the
> two ways to read each ball joint's rotation as three angles, which pair do you take, and
> where on the swivel circle do you put the knee. This file makes both — three different
> ways, depending on whether anyone is enforcing joint limits.

**Needs** — [`limb.h`](limb.h.md) · [`Dof7control.h`](Dof7control.h.md) · [`eulersolver.h`](eulersolver.h.md) · [`aint.h`](aint.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2. With limits off — the shipping path — it is a handful of matrix
multiplies and two Euler decompositions, allocating nothing. With limits on it builds
interval sets and therefore allocates.

## Purpose

[`Dof7control.cpp`](Dof7control.cpp.md) works in matrices and knows no joint limits.
The skeleton wants angles and has limits. This file is the conversion, and it is where
"which of the two solution families" and "which swivel angle" are decided.

Three decision procedures live here and only the first is used in shipping play:

- **Limits off.** Take the swivel angle the caller supplies, and for each ball joint take
  whichever family sits nearer its authored limits. No intervals, no analysis.
- **Limits on, caller has no preference.** Compute the four feasible swivel-angle sets,
  union them, take the midpoint of the largest continuous stretch, and use whichever
  family that stretch came from.
- **Limits on, caller has a preferred swivel angle.** Try it; if it violates something, try
  any nearby singularity; failing that, move to the nearest point of the nearest feasible
  set.

## State

```text
RECORD Limb
  solver       : ChainSolver        # the geometry, from Dof7control
  euler1       : Convention         # how to read the root ball joint's rotation
  euler2       : Convention         # ... and the tip's
  min, max     : list<real> (7)     # authored joint limits, indexed 0..6
  jt_limits    : list<Arc>  (7)     # the same, as arcs on the circle
  check_limits : bool               # set by the last goal posed
  mode         : {position_only, position_and_orientation}
  hinge_angle  : real               # index 3; solved with the goal, never chosen
  PSI          : list<ArcSet> (4)   # feasible swivel angles per family combination
  singular     : list<real> (4)     # swivel angles where a family decomposition breaks
```

**Invariants** —

- Index 3 — the hinge — is not part of either ball joint and is never produced by an
  Euler decomposition. It is solved with the goal and copied into every answer unchanged.
- `PSI` is meaningful only when the goal was posed with limits on; the solve paths that
  read it are exactly the paths that only run in that case.
- **The joint index order is the reverse of the Euler convention's order.** The Euler
  machinery yields its three angles outermost-first; the skeleton indexes them
  root-first. Every extraction therefore exchanges the first and third angle immediately
  after decomposing. A rebuild that aligns the two orders deletes four identical swaps and
  one class of very confusing bug.
- Limits are half-open arcs in a single positive turn, so a limit that spans zero wraps.
  Every containment test goes through the arc type for that reason, never through a pair
  of comparisons.

## `init`

**Contract** — store the two link transforms and the two Euler conventions in the
geometric solver, record the seven limits both as numbers and as arcs, and start with
limits disabled and no goal posed. Runs once per limb.

**Notes** — the limits are kept in both forms because the arcs answer containment and the
raw numbers answer "put this angle in range", which needs the ends as ordinary reals.

## `SetGoal` · `SetGoalPos`

**Contract** — pose a goal on the geometric solver, then check the hinge angle against its
own limit. Failure of either means no solution and the caller keeps the previous pose.
With limits on, additionally compute the feasible swivel-angle sets. Returns success.

```text
FUNCTION set_goal(G, limits_on) -> bool
  hinge_angle <- solver.set_goal(G)   OR RETURN false
  # The hinge is one degree of freedom and it is already determined; if it violates
  # its limit there is nothing to trade against, so the whole goal fails here.
  IF NOT hinge_in_range(hinge_angle) RETURN false
  mode         <- position_and_orientation
  check_limits <- limits_on
  IF limits_on
    PSI <- feasible_swivel_sets_for_both_ball_joints()
  RETURN true

FUNCTION hinge_in_range(v) -> bool
  IF NOT jt_limits[3] contains v RETURN false
  # The arc test works on the circle; the caller wants a number inside [min, max].
  # Shift by whole turns until it is.
  IF v < min[3] THEN v <- v + one_turn
  IF v > max[3] THEN v <- v - one_turn
  RETURN true
```

**Notes** — the hinge check is the *only* joint limit the shipping path enforces, because
it happens before the limits-on branch. That is worth knowing: a leg can bend its knee
backwards only if the authored knee limit permits it, but its hip and ankle are
unconstrained in normal play. The engine compensates by widening the authored limits by
about a radian on the way in and then relying on the goal being a small perturbation of a
legal pose.

## `extract_s1` · `extract_s1s2` — choosing a family

**Contract** — decompose one or both ball joint rotations into three angles each, with no
guarantee that the result is legal. Both families are computed and the one that violates
its limits *least* is kept.

```text
FUNCTION extract_ball_joint(R, convention, limit_arcs, min, max) -> (a0, a1, a2)
  (f1, f2) <- decompose_both(convention, R)
  swap the first and third angle of each          # convention order -> joint order
  best <- family with the smaller total violation:
            for each of the three angles, the arc's DISTANCE if positive, else 0
  FOR i IN 0..2
    best[i] <- put_in_range(min[i], max[i], best[i])
  RETURN best

FUNCTION put_in_range(low, high, v) -> real
  # v is in one positive turn; the limits may not be. Try v and v minus a full turn,
  # and keep whichever lands inside — or, if neither does, whichever misses by less.
  IF low <= v <= high RETURN v
  w <- v - one_turn
  IF low <= w <= high RETURN w
  RETURN the one whose distance to the nearer end is smaller
```

**Notes** — *least violation*, not *no violation*. This is the entire joint-limit policy
of the shipping engine: it never refuses a pose, it picks the less bad of two readings and
moves on. The signed distance from [`aint.cxx`](aint.cxx.md) is used with its negative
values discarded, so a family that is comfortably inside its limits scores the same zero
as one that is barely inside — only violations are summed. That is deliberate: preferring
"deeper inside" would bias every pose toward the middle of the joint's range and flatten
the animation.

The per-family variants do the same without the choice, for the paths where the family has
already been fixed by the interval analysis.

## `get_R1psi` · `get_R1R2psi` — the feasible swivel sets

**Contract** — with a goal posed, compute for each ball joint the set of swivel angles
that keep all three of its angles legal, per family, then intersect across the three
angles and across the two joints. Produces two sets for a position-only goal and four for
a full goal. Only reachable with limits on.

```text
FUNCTION feasible_swivel_sets_for_both_ball_joints() -> ArcSet[4]
  (C, S, O, C2, S2, O2) <- solver.both_as_functions_of_psi()

  # Note the limits are handed over REVERSED - convention order again.
  root <- SwivelDecomposer(euler1, C,  S,  O,  limits[2],limits[1],limits[0])
  tip  <- SwivelDecomposer(euler2, C2, S2, O2, limits[6],limits[5],limits[4])

  (r1, r2) <- root.psi_ranges()        # per family, per angle: 3 sets each
  (t1, t2) <- tip.psi_ranges()
  singular <- root.singularities() ++ tip.singularities()

  # A family is feasible only where all three of its angles are.
  root_f1 <- r1[0] ∩ r1[1] ∩ r1[2]
  root_f2 <- r2[0] ∩ r2[1] ∩ r2[2]
  tip_f1  <- t1[0] ∩ t1[1] ∩ t1[2]
  tip_f2  <- t2[0] ∩ t2[1] ∩ t2[2]

  IF root_f1 and root_f2 are both empty RETURN four empty sets
  RETURN [ root_f1 ∩ tip_f1, root_f1 ∩ tip_f2,
           root_f2 ∩ tip_f1, root_f2 ∩ tip_f2 ]
```

**Notes** — the four sets are the four *family combinations*, and they are genuinely
distinct problems: a swivel angle may be legal with the hip read one way and the ankle the
other, and illegal with both read the same way. Any solve that uses these sets must
therefore record which of the four it came from and decompose accordingly.

The singular swivel angles from both joints are concatenated into one array whose length
is the sum. That array is fixed at four entries and each joint can contribute two, so it
is exactly full; a rebuild should size it from the contributions rather than assume.

## `Solve` — no preferred swivel angle

**Contract** — solve the posed goal, letting the limb choose the swivel angle. Writes seven
angles; optionally reports the angle chosen and the hinge position it puts the knee at.
Returns success.

```text
FUNCTION solve(x) -> bool
  x[3] <- hinge_angle
  IF NOT check_limits
    RETURN solve_by_angle(0, x)              # limits off: swivel zero, take it as it comes

  family <- choose_largest_range(PSI)        # sets the swivel angle as a side effect
  IF family exists
    decompose both ball joints in that family at that angle INTO x
  ELSE
    # No feasible stretch at all. The interval analysis is unreliable AT a
    # singularity, so try each one directly before giving up.
    family <- try each singular swivel angle, keep the first that satisfies every limit
  RETURN family exists

FUNCTION choose_largest_range(sets) -> family
  # Union all the sets, with a MERGE TOLERANCE of about 0.05 radians - twenty times
  # the set type's own loose tolerance. Two feasible stretches separated by less than
  # three degrees are treated as one, because the midpoint of a hairline-split
  # stretch is arbitrary and moves violently frame to frame.
  all <- union of every set, merged at 0.05
  a <- the largest arc in `all`, or RETURN none
  swivel <- midpoint of a
  RETURN whichever input set contains that midpoint
      OR, if rounding put it just outside all of them, whichever is nearest
```

**Notes** — the midpoint of the largest feasible stretch is the "most comfortable" pose:
the swivel angle furthest from any joint reaching a limit. It is a reasonable default and
it is the wrong thing for foot placement, which is why the engine does not use this entry
point — a knee that relocates to the most comfortable position every time the ground
changes looks like a malfunction. See the next section.

## `SolveByAngle` · `SolveByPos` — the path the engine uses

**Contract** — solve the posed goal at a *given* swivel angle, or at the swivel angle
corresponding to a given knee position. Writes seven angles. With limits off this always
succeeds and simply honours the requested angle; with limits on it may move the angle, and
reports the angle it settled on.

```text
FUNCTION solve_by_angle(psi, x) -> bool
  normalize psi into one positive turn
  x[3] <- hinge_angle

  IF NOT check_limits
    decompose both ball joints at psi, choosing each family by least violation
    RETURN true                              # always succeeds

  # Limits on: prefer the requested angle, then a nearby singularity, then the
  # nearest legal angle.
  IF trying psi outright satisfies every limit
    RETURN true
  FOR EACH singular angle within one degree of psi
    IF trying it satisfies every limit
      psi <- it; RETURN true
  family <- nearest feasible set to psi, moving psi to its nearest boundary
  IF family exists
    decompose in that family at the moved psi
  RETURN family exists
```

**Notes** — the *one degree* window around a singularity is the file's own admission that
the interval analysis is untrustworthy there. Near a singularity the feasible sets are
computed from ratios whose denominator is vanishing, and the honest answer is to evaluate
the pose directly rather than believe the intervals. A rebuild should keep the structure —
try, then probe the known-bad points, then fall back — even if its tolerance differs.

The position variant converts through the geometric solver and delegates. This is the
engine's actual call: it computes where the animator put the knee, re-expresses it against
the *new* hip-to-foot direction, and hands the position in. With limits off, that means
the shipping solve is: take the animator's knee, take the goal, produce seven angles,
never fail.

## `KneeAngle`

**Contract** — the swivel angle corresponding to a knee position, measured against an
arbitrary goal position that has **not** been posed as a goal. Recomputes the circle for
that position, converts, and restores the previous state.

**Notes** — this exists for exactly one caller: the engine asking "what swivel angle does
the animation's own pose correspond to" before it has decided what goal to set. It
temporarily marks the limb as having a goal so that the conversion's own guard passes.
That is a hack around a guard that only exists to catch programmer error; a rebuild with
a solver that does not require a posed goal to evaluate its circle needs none of it.

## `InLimits` · `ForwardKinematics`

**Contract** — are all seven angles inside their arcs; and compose seven angles back into
the tip's transform, in the chain's order: root ball joint, root link, hinge about the
*y* axis, hinge link, tip ball joint. The forward pass exists to verify the inverse.

**Notes** — the forward pass reverses the angle triples on the way in, mirroring the swap
on the way out. It is the clearest statement in the directory of the index-order mismatch.

## `Debug`

**Contract** — write the two swivel-parameterized coefficient matrices and their limits to
two text files, for offline analysis of a pose that misbehaves. Present, unused, and worth
having in a rebuild for the same reason: the closed-form analysis is hard to debug from
inside a frame.
