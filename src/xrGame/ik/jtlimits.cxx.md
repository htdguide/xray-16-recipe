# src/xrGame/ik/jtlimits.cxx

> Turns one joint's authored angular range into the set of swivel angles that respect it.
> The joint angle is a continuous function of the swivel angle except where it jumps; find
> the jumps, find the crossings of the two limits, and test the midpoint of each resulting
> stretch. That is the whole method, applied twice with different arithmetic.

**Needs** — [`jtlimits.h`](jtlimits.h.md) · [`aint.h`](aint.h.md) · [`eqn.h`](eqn.h.md)
**Used by** — reached through its declarations in [`jtlimits.h`](jtlimits.h.md); callers name that, not this file.
**Tier floor** — T2. Scalar trigonometry producing interval sets; allocates through the
interval set.

## Purpose

This is the file the `ik` chapter's README calls "genuinely elegant and switched off".
Given a goal, [`Dof7control.cpp`](Dof7control.cpp.md) can express every entry of the hip's
and the ankle's rotation matrices as a sinusoid in the swivel angle ψ. Each joint angle is
therefore an explicit function of ψ, and asking "for which ψ does this joint stay within
its authored range" is a question with an exact answer — a set of arcs, computed without
sampling and without iteration.

The shipping engine never asks. It passes *limits off* on every solve and relies on the
goal being a small perturbation of a pose an animator already made legal. Everything here
runs only behind a debug draw flag. Build it last, or not at all.

## State

```text
RECORD SimpleJoint              # sin(theta) = alpha*cos(psi) + beta*sin(psi) + xi
  curve    : Curve
  arc      : Arc                # the joint's authored range, as an arc on the circle
  sin_low  : real               # sin of each end of the arc, precomputed: the crossing
  sin_high : real               # tests need them on every call

RECORD ComplexJoint             # theta recovered as a ratio:
                                #   sin(theta)*s = curve_sin(psi)
                                #   cos(theta)*s = curve_cos(psi)
                                #   s            = curve_factor(psi)
  curve_sin, curve_cos : Curve
  curve_factor         : Curve
  ratio_derivative     : Curve  # numerator of d(theta)/d(psi); see below
  arc                  : Arc
  tan_low, tan_high    : real   # tangent of each arc end, nudged off the quarter turns
```

**Invariants** — the two precomputed sines, and the two precomputed tangents, must be
recomputed whenever the arc changes. The type enforces this by routing every write to the
arc through a setter that also updates them; a rebuild should make the arc and its
derived values one value written together.

`ratio_derivative` is not a derivative of any single curve: it is the numerator of
`(sin' · cos − sin · cos')`, which for two sinusoids is itself a sinusoid, with
coefficients built from cross products of the other two curves' coefficients. Stating it
as a curve is what lets the joint's stationary points be found by the same root-finder as
everything else.

## Why two kinds of joint

A three-degree-of-freedom joint decomposed into Euler angles yields one angle that appears
*alone* in some matrix entry — its sine is read straight off — and two that appear only
*multiplied by* the first one's sine or cosine. The first is the simple joint. The other
two are complex: their angle is `atan2` of two entries, and the common factor cancels —
except where the factor is zero, which is the singular pose where the two outer rotations
become the same rotation. That is the entire structural difference, and it is why the
complex joint needs singularities and the simple one only needs discontinuities.

## `SimpleJoint.theta` · `theta_d`

**Contract** — the joint angle for a swivel angle, in a chosen family; and its derivative.
Family one takes the inverse sine into the quarter turns either side of zero, family two
into the other half. The derivative is `curve'(ψ) / √(1 − curve(ψ)²)` for family one and
its negation for family two.

**Notes** — the derivative's denominator vanishes exactly where the joint is at a quarter
turn, and the code answers by averaging the derivative a small step either side, widening
that step tenfold per level of recursion. It is a finite-difference patch over a genuine
infinity and it is only ever used to orient a search, never to produce a pose. A rebuild
may return a large signed value instead.

## `SimpleJoint.Solve`

**Contract** — the swivel angles at which the joint reaches a given angle, in a chosen
family. Zero, one or two. Returns none immediately if the requested angle does not lie in
the family's half of the circle — family one covers the quarter turns either side of zero,
family two the rest. The caller passes the requested angle's sine alongside it, already
computed.

**Notes** — the half-circle test before the solve is what keeps the two families from
reporting each other's solutions. Without it, both families answer every query and the
feasible sets come out as the whole circle.

## `SimpleJoint.Discontinuity`

**Contract** — the swivel angles at which the family-one joint angle jumps. Family two
never jumps and reports none.

**Notes** — family one is the inverse sine into the quarter turns either side of zero; it
is continuous everywhere except where its argument crosses zero, because there the
principal branch changes sign discontinuously as an *angle on the circle* (a value just
below zero normalizes to just below a full turn). So the discontinuities are exactly the
curve's roots, already memoized. This is the observation the whole file turns on: the
function is piecewise continuous, and the pieces are cut by things already known.

## `SimpleJoint.PsiLimits`

**Contract** — the headline operation. Produces two sets of arcs — one per family — of the
swivel angles for which this joint stays within its authored range. Clears both outputs
first. Costs a handful of transcendental calls; no iteration, no sampling beyond one
midpoint test per stretch.

```text
FUNCTION psi_limits(joint) -> (family1 : ArcSet, family2 : ArcSet)
  eps <- loose_tolerance / 5          # see note

  # Family 1 is piecewise continuous; cut the circle at its jumps.
  cuts <- [eps] + sorted(joint.discontinuities(family 1)) + [one_turn - eps]
  FOR EACH consecutive pair (a, b) IN cuts
    IF b - a < 2*eps CONTINUE         # the piece is narrower than the tolerance
    clip(family 1, from a + eps, to b - eps, joint.arc, INTO family1)

  # Family 2 is continuous everywhere; one piece, the whole circle.
  clip(family 2, from eps, to one_turn - eps, joint.arc, INTO family2)

FUNCTION clip(family, psi0, psi1, arc, INTO out)
  # Where does theta(psi) cross either end of the joint's range?
  boundaries <- [psi0]
             + sorted(solve(family, arc.low)  ++ solve(family, arc.high)
                      restricted to [psi0, psi1])
             + [psi1]
  # Between two consecutive crossings theta cannot leave the range, so one test
  # decides the whole stretch. This is the step that replaces sampling.
  FOR EACH consecutive pair (a, b) IN boundaries
    IF theta(family, midpoint(a, b)) lies within arc
      add the arc (a, b) TO out
```

**Invariants** — the joint's arc may wrap through zero. A wrapping arc is handled by
running the clip twice, once against `[low, full turn]` and once against `[0, high]`, and
letting the interval set's merging rejoin them. This is the same split-and-recombine used
throughout [`aint.cxx`](aint.cxx.md).

**Notes** — the epsilon is the interval set's *loose* tolerance divided by five, which
makes it five times larger than the tight one and five times smaller than the merge
tolerance. It is inset at both ends of every stretch so that no boundary angle is ever
evaluated exactly at a discontinuity. **The specific factor of five has no discoverable
derivation** and reads as a value someone settled on; what matters is the ordering
tight ≪ eps ≪ loose.

## `ComplexJoint.theta` · `theta_d` · `CritPoints`

**Contract** — the joint angle is `atan2(curve_sin(ψ), curve_cos(ψ))` for family one and
the same with both arguments negated — a half turn away — for family two. The derivative
is the cross term over `1 − factor(ψ)²`, and it is the *same* for both families, unlike
the simple joint's, which flips sign. Critical points are the roots of the stored
derivative-numerator curve.

**Notes** — that the two families share a derivative is not obvious and is easy to
"correct" into a bug: the families differ by a constant half turn, and a constant
differentiates to zero.

The derivative near a singularity is again patched by averaging either side. Here the code
also checks that the two sides agree in sign and reports zero when they do not, which is
the honest answer at a pole.

## `ComplexJoint.Singularities`

**Contract** — the swivel angles at which the shared factor reaches ±1, i.e. where the
other factor vanishes and the ratio is undefined. At most two, sorted. Computed by solving
the factor curve for `1 − ε` and `−1 + ε` rather than for exactly ±1, and averaging the
two roots when a tangency yields a pair.

**Notes** — the deliberate offset by a small epsilon is the point. At exactly ±1 the curve
is tangent and the two roots have merged; solving there is numerically hopeless, so the
solve is done just inside, where two distinct roots exist, and their midpoint is taken as
the singularity. A rebuild that solves at exactly ±1 gets either no answer or a very noisy
one.

The comment that singularities cannot be found by looking at the derivative is worth
keeping: at a pole the derivative is not merely zero, it is undefined, and a root-finder
on it silently misses them.

## `ComplexJoint.Solve` · `Solve2`

**Contract** — the swivel angles at which the joint reaches a given angle, for one family
or split across both. The caller passes the tangent of the requested angle alongside it.

```text
FUNCTION solve_ratio(joint, v, tan_v) -> list<real>
  # Three cases, because the general reduction divides by a cosine that can vanish.
  IF v is at a quarter turn           RETURN roots of curve_cos
  IF v is at zero or a half turn      RETURN roots of curve_sin
  # General: tan(theta) = v  =>  curve_sin - tan(v)*curve_cos = 0, itself a sinusoid.
  RETURN roots of (curve_sin - tan_v * curve_cos)

FUNCTION solve(joint, family, v, tan_v) -> list<real>
  # The tangent cannot tell an angle from its opposite, so half the roots belong to
  # the other family. Evaluate each and keep the ones that actually reproduce v.
  RETURN [ p IN solve_ratio(joint, v, tan_v) WHERE theta(family, p) equals v ]
```

**Notes** — the two-family variant assigns each root to whichever family reproduces the
requested angle, and *complains to the log* when a root reproduces neither. That third
case happens near a singularity and is the file's own admission that the analysis degrades
there. The complaint is why the tangents of the arc's ends are nudged away from the
quarter turns at construction — the nudge is about a ten-thousandth of a radian, chosen to
be far larger than the comparison tolerance and far smaller than anything visible.

## `ComplexJoint.PsiLimits`

**Contract** — the same headline operation as the simple joint's, with the same shape: cut
the circle, collect crossings, test each stretch's midpoint, emit arcs. Two differences,
both forced by the ratio form:

- The cut set is the union of the **singularities supplied by the caller** and the roots
  of the sine curve. Singularities come from outside because the two complex joints of one
  spherical joint share them and computing them twice would be waste — and, more
  importantly, would risk the two joints cutting the circle at slightly different places.
- Crossings are gathered for both families in one pass, since the ratio solve naturally
  produces both.

## `mytan`

**Contract** — the tangent, with its argument pushed a small distance off a quarter turn
whenever it lands within a tolerance of one. Returns a large finite number instead of an
infinity, on the correct side.

**Notes** — the nudge is one ten-thousandth of a radian and the detection tolerance is one
hundred-thousandth, so the nudge always moves the argument strictly further from the pole
than the tolerance that detected it. That ordering is the decision; the magnitudes are
free within it.
