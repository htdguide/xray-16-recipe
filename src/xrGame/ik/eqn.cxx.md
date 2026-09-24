# src/xrGame/ik/eqn.cxx

> Roots and stationary points of a sinusoid with an offset, memoized — because the joint
> limit analysis asks for the same three answers about the same curve over and over.

**Needs** — [`eqn.h`](eqn.h.md)
**Used by** — reached through its declarations in [`eqn.h`](eqn.h.md); callers name that, not this file.
**Tier floor** — T2. Scalar arithmetic with a cached result behind a mutable-through-const
back door, which is an artifact of the language and not a decision.

## Purpose

`α·cos ψ + β·sin ψ + ξ` is the same curve as `√(α²+β²)·cos(ψ − atan2(β,α)) + ξ`: a single
cosine with an amplitude, a phase and an offset. Once it is seen that way, every question
about it is one line of trigonometry. This file is that observation, written down once, so
that [`jtlimits.cxx`](jtlimits.cxx.md) and [`eulersolver.cxx`](eulersolver.cxx.md) can ask
for roots and crossings without rederiving it.

## State

```text
RECORD Curve                    # alpha*cos(psi) + beta*sin(psi) + xi
  alpha, beta, xi : real
  amplitude_sq    : real        # alpha^2 + beta^2, fixed at construction
  phase           : real        # atan2(beta, alpha), fixed at construction

  # memo, filled on first ask and never invalidated except by a Reset
  computed        : set<{roots, crits}>
  roots           : list<real>  # at most 2
  crits           : list<real>  # at most 2
```

**Invariants** — `amplitude_sq` and `phase` must be recomputed whenever the coefficients
change, and the memo must be discarded at the same moment. The single reset operation does
both; nothing else may write a coefficient. In the original this invariant is enforced
only by convention, and the memo is written through pointers-to-self that exist solely to
mutate a logically-constant object. A rebuild with interior mutability, or with the memo
computed eagerly, deletes all of that.

The curve is only meaningful where `|α·cos ψ + β·sin ψ + ξ| ≤ 1`, because every caller is
feeding the result to an inverse sine or cosine. Nothing enforces it; the solvers clamp.

## `eval` · `deriv`

**Contract** — the curve's value and its slope at an angle. Both inline. The derivative is
`β·cos ψ − α·sin ψ` — the same shape with the coefficients rotated a quarter turn — so it
reuses the evaluator with its arguments permuted rather than having code of its own.

**Notes** — the evaluator computes `cos ψ` and then derives `sin ψ` as
`±√(1 − cos²ψ)`, choosing the sign by whether ψ is in the upper or lower half of the turn.
That is one transcendental call plus a square root instead of two transcendental calls.
The trade was worth making on the hardware of the time and is probably not worth making
now; it is arithmetically exact either way. What a rebuild must keep is the *argument
normalization into a single positive turn* that precedes it, since the half-turn test is
meaningless without it.

## `roots`

**Contract** — the angles where the curve crosses zero. Zero, one or two of them, in
increasing order, memoized after the first call.

```text
FUNCTION roots(curve) -> list<real>
  RETURN crossings(curve, 0)
```

## `solve`

**Contract** — the angles where the curve takes a given value. Zero, one or two, in
increasing order. Not memoized, since the value varies per call.

```text
FUNCTION crossings(curve, v) -> list<real>
  # Solve amplitude*cos(psi - phase) = v - xi.
  d <- amplitude_sq - (v - xi)^2
  IF d < 0 RETURN []                        # the line misses the curve entirely

  spread <- arctan2(sqrt(d), v - xi)
  IF |spread| is below 1e-6
    RETURN [ phase ]                        # tangent: the two solutions have merged
  RETURN sorted([ phase + spread, phase - spread ])
```

**Notes** — this is the same solver as `solve_trig1` in
[`mathTrig.cpp`](mathTrig.cpp.md), with three differences that are all deliberate:
the amplitude and phase come from the record instead of being recomputed; the answers are
returned **sorted**, which every caller here depends on because they are about to be used
as interval boundaries; and the near-tangent case is decided by an *absolute* tolerance on
the spread rather than a relative one on the discriminant. The absolute test is the weaker
of the two, and it is used here because the caller can tolerate an extra near-duplicate
root where the other caller could not tolerate a spurious failure.

The original keeps a consistency check that runs both solvers and compares them; it is
compiled out.

## `crit_points`

**Contract** — the angles where the curve is stationary. One or two, memoized. These are
the maximum and the minimum, so they bracket the curve's range and are where a monotone
analysis has to be cut.

```text
FUNCTION crit_points(curve) -> list<real>
  # The derivative is beta*cos(psi) - alpha*sin(psi); its zeros are the curve's
  # stationary points. This must be solved from the derivative's OWN coefficients,
  # not through the cached amplitude and phase, which belong to the curve.
  RETURN solve( beta*cos(psi) + (-alpha)*sin(psi) = 0 )
```

**Notes** — the comment in the original that the cached form "CANNOT" be used here is the
one warning in the file worth carrying forward. The cache describes the curve; the
critical points are a property of its derivative, whose phase is a quarter turn away.

## What is not here

Two further operations are written out in the source and disabled: clipping the curve
against a band `low ≤ f(ψ) ≤ high` to produce the set of ψ that satisfy it, and
partitioning the circle into the regions where the curve lies above and below a value.
Both were superseded by the equivalent routines in [`jtlimits.cxx`](jtlimits.cxx.md),
which need the same walk but must also apply an inverse trigonometric function and pick a
solution family at each step. A rebuild should not restore them; it should notice that the
survivors could be expressed in terms of them and probably should be.
