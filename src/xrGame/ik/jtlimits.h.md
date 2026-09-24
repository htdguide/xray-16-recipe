# src/xrGame/ik/jtlimits.h

> Declares the two kinds of joint whose feasible swivel angles can be computed in closed
> form, and the inverse trigonometry that distinguishes their solution families.

**Needs** — [`eqn.h`](eqn.h.md)
**Used by** — [`eulersolver.cxx`](eulersolver.cxx.md) · [`eulersolver.h`](eulersolver.h.md) · [`jtlimits.cxx`](jtlimits.cxx.md)
**Tier floor** — T2. Records holding a few curves and an arc.

## Purpose

Declares the surface implemented in [`jtlimits.cxx`](jtlimits.cxx.md), which carries the
substance, and carries the four quadrant-restricted inverse functions inline because they
are one call each.

## Exported units

**Four inverse trigonometric functions, each restricted to a half of the circle** — arcsine
into the quarter-turns either side of zero and into the two beyond, arccosine into the
first half turn and into the second. An argument outside `[-1, 1]` is *clamped* to the
boundary rather than rejected, because at this point in the pipeline such an argument is
always a rounding excursion and never a real infeasibility. These four are the concrete
meaning of "solution family": family one uses the first of each pair, family two the
second.

**A tag** saying whether a joint's equation gives the sine or the cosine of its angle.
Only the sine case is implemented anywhere in the file; the cosine case prints a complaint
and returns nothing. That is honest to reproduce: the Euler conventions the solver
actually uses never produce a cosine-form joint.

**A simple joint** — one whose angle satisfies `sin θ = α·cos ψ + β·sin ψ + ξ`. It holds
that curve and the joint's authored arc. Its operations: the joint angle for a given
swivel angle in each family, the derivative of that, where the angle can jump, the
swivel angles that put the joint exactly at a given value, and — the point of the whole
type — **the set of swivel angles for which the joint stays inside its arc**, one set per
family.

**A complex joint** — one whose angle is recovered from a *ratio*, because its equations
carry a factor of the simple joint's sine or cosine:
`sin θ·s = …`, `cos θ·s = …`, `s = …`, three curves in all. It additionally holds the
derivative of the ratio, and the tangents of its arc's two ends, precomputed because they
are needed on every crossing test and because the tangent is singular at the quarter
turns and must be nudged aside. Same operations as the simple joint, plus its critical
points and its **singularities** — the swivel angles where the shared factor vanishes and
the ratio is undefined. Those are handed back to the caller so they can be computed once
and reused across the two complex joints of one spherical joint.
