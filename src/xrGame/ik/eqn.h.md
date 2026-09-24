# src/xrGame/ik/eqn.h

> Declares the one function shape the whole joint-limit analysis is built from:
> `α·cos ψ + β·sin ψ + ξ`.

**Needs** — [`aint.h`](aint.h.md)
**Used by** — [`eqn.cxx`](eqn.cxx.md) · [`jtlimits.cxx`](jtlimits.cxx.md) · [`jtlimits.h`](jtlimits.h.md)
**Tier floor** — T2. A small record with memoized scalar results.

## Purpose

Declares the surface implemented in [`eqn.cxx`](eqn.cxx.md), and carries the evaluation
and its derivative inline because they are one expression each.

The reason this shape gets a type of its own rather than three loose floats: every entry
of the hip's rotation matrix, as a function of the swivel angle ψ, has exactly this form —
so every question the joint-limit machinery asks ("where does this matrix entry cross a
value", "where is it stationary") is a question about this one curve, asked thousands of
times per analysed goal. Naming it lets the answers be cached.

## Exported units

- A free evaluator of `α·cos x + β·sin x`, which computes the sine from the cosine and a
  sign chosen by which half of the turn `x` falls in — trading a transcendental call for a
  square root and a branch.
- **The curve record** itself: its three coefficients, and two quantities derived from
  them once at construction — `α² + β²` and `atan2(β, α)` — which are what the root and
  critical-point solvers actually need.
- Reset, to re-point an existing record at new coefficients without reallocating.
- Evaluate at an angle, and the derivative at an angle — which is the same curve with its
  coefficients rotated, so it needs no separate code.
- Critical points: where the derivative vanishes. One or two.
- Roots: where the curve crosses zero. Zero, one or two.
- Solve: where the curve crosses a given value. Zero, one or two.

Three further operations — clipping the curve against a band, and partitioning the circle
into the regions above and below a value — are declared in comment form only and are not
built. Their work is done in [`jtlimits.cxx`](jtlimits.cxx.md) instead.
