# src/xrGame/ik/mathTrig.cpp

> Every joint in the chain is eventually one of two equations; this solves both in closed
> form, and takes care to report *how many* answers there are, because the count is what
> tells the solver whether it is near a degenerate pose.

**Needs** — [`mathTrig.h`](mathTrig.h.md)
**Used by** — reached through its declarations in [`mathTrig.h`](mathTrig.h.md); callers name that, not this file.
**Tier floor** — T3. Four scalar functions. It is in this directory rather than the
engine's own math layer only because it is vendored with the solver.

## Purpose

The chain solve never iterates. Every angle it produces comes out of one of the two
equations here, and the whole "no convergence criterion, fixed microsecond cost" property
of the feature rests on that. This file is small and load-bearing out of proportion to
its size.

## State

Stateless.

## `solve_trig1`

**Contract** — solves `a·cos θ + b·sin θ = c` for θ, writing up to two answers and
returning how many exist: two in the general case, one when the line is tangent to the
circle, zero when there is no solution. Radians, unnormalized — the caller may receive an
angle outside a single turn and is expected to cope. No allocation.

**Invariants** — the two answers are `atan2(b,a) ± atan2(√(a²+b²−c²), c)`; they coincide
exactly when the second term vanishes.

```text
FUNCTION solve_trig1(a, b, c) -> list<real>
  d <- a*a + b*b - c*c

  IF d < 0
    # Genuinely no solution, OR a tangent case that rounding pushed negative.
    # Distinguishing them is the whole reason this branch exists: relative to the
    # magnitude of the inputs, a d that is below one part in a million is zero.
    IF |d| / (|a*a| + |b*b| + |c*c|) < 1e-6
      RETURN [ 2 * arctan( -b / (-a - c) )]      # the single tangent solution
    RETURN []                                     # no solution

  spread <- arctan2(sqrt(d), c)
  base   <- arctan2(b, a)
  IF spread is zero within tolerance
    RETURN [ base ]
  RETURN [ base + spread, base - spread ]
```

**Notes** — the relative-tolerance test is the only numerically interesting line in the
file, and it is the difference between a leg that solves and a leg that reports failure
once every few hundred frames when the knee happens to be straight. An absolute tolerance
would be wrong: the inputs are squared lengths in metres, so their scale varies with the
creature.

The tangent solution is written in a form that avoids dividing by zero at `c = −a`
rather than the obvious one; it is the half-angle identity and a rebuild may use whichever
form its language's `atan2` is happiest with.

## `solve_trig2`

**Contract** — solves the *pair*
`a·cos θ − b·sin θ = c` and `a·sin θ + b·cos θ = d` for the single θ that satisfies both,
in radians. There is always exactly one, so there is no failure case and no count.

```text
FUNCTION solve_trig2(a, b, c, d) -> real
  RETURN arctan2(a*d - b*c, a*c + b*d)
```

**Notes** — the pair says "the rotation by θ carries the vector (a,b) onto (c,d)". Two
constraints, one unknown, and consistency guaranteed by the geometry that produced them —
so the answer is a single quadrant-correct arctangent and never needs a solution count.
This is the shape every *orientation* recovery in the solver takes, as opposed to the
position recoveries, which take the previous function's shape.

## `myacos` · `myasin`

**Contract** — inverse cosine and inverse sine returning *both* angles in a full turn
whose cosine or sine is the argument, written into a two-element output, with the count
returned. Zero when the argument is outside `[-1, 1]`. One when the two answers coincide —
at the extremes of the range. The first answer is always the principal one, normalized to
a signed half-turn.

```text
FUNCTION two_valued_arccos(x) -> list<real>
  IF |x| > 1 RETURN []
  a <- normalize_signed(arccos(x))
  IF a is zero within tolerance RETURN [a]
  RETURN [a, -a]                    # cosine is even: ±a share a cosine

FUNCTION two_valued_arcsin(x) -> list<real>
  IF |x| > 1 RETURN []
  a <- normalize_signed(arcsin(x))
  IF a is zero within tolerance RETURN [a]
  IF a > 0  RETURN [a,  half_turn - a]   # sine is symmetric about a quarter turn
  ELSE      RETURN [a, -half_turn - a]
```

**Notes** — these two functions are why the solver has "families". A joint angle recovered
from a sine has two candidates a half-turn apart in the *other* two joints of the same
spherical joint; picking between them is a decision made further up, in
[`limb.cxx`](limb.cxx.md), and it is picked by which candidate sits closer to the
authored joint limits.

An argument outside `[-1, 1]` is reported as no solution here, but the same situation is
*clamped* rather than rejected in [`jtlimits.cxx`](jtlimits.cxx.md). The two conventions
coexist because these are the entry points used when a genuine failure is possible, and
those are the ones used when the argument is known to be a rounding excursion past the
boundary.

## `law_of_cosines`

**Contract** — given side lengths `a`, `b` and the side `c` opposite the angle sought,
returns that angle and success — or failure if no triangle has those three sides. In the
solver's terms: given the two link lengths and the required root-to-tip distance, the
knee's bend. Declared inline in the header; it is one expression.

```text
FUNCTION law_of_cosines(a, b, c) -> optional<real>
  t <- (a*a + b*b - c*c) / (2*a*b)
  IF |t| > 1 RETURN none          # the goal is unreachable: no such triangle
  RETURN arccos(t)
```

**Notes** — the failure case is the unreachable-goal case, and it is the reason the goal
is pulled back to just inside the leg's own length *before* the solve rather than being
rejected here. By the time the solve runs, this function is not allowed to fail, and if
it does the frame silently keeps the previous pose. See the reachability clamp in
[`Dof7control.cpp`](Dof7control.cpp.md).
