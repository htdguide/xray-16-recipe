# src/xrGame/ik/mathTrig.h

> Declares the two trigonometric equations every closed-form step of the solver reduces
> to, plus the law of cosines that fixes the knee.

**Needs** — _(none)_
**Used by** — [`Dof7control.cpp`](Dof7control.cpp.md) · [`mathTrig.cpp`](mathTrig.cpp.md)
**Tier floor** — T3. Scalar trigonometry.

## Purpose

Declares the surface implemented in [`mathTrig.cpp`](mathTrig.cpp.md), and carries two
small solvers inline because they are one expression each.

## Exported units

- `solve_trig1` — all solutions of `a·cos θ + b·sin θ = c`. One or two.
- `solve_trig2` — the single solution of the pair `a·cos θ − b·sin θ = c`,
  `a·sin θ + b·cos θ = d`.
- `myacos`, `myasin` — the inverse functions returning *both* angles in a full turn, not
  the principal one. The second solution is what makes two solution families exist.
- `law_of_cosines` (inline) — given three side lengths, the angle between the first two,
  or a failure if no triangle has those sides. This is the knee.
- `iszero` (inline) — the file's tolerance: a value counts as zero when its square is
  below one part in a million, i.e. a magnitude below about 0.001. Used to decide when a
  pair of solutions has collapsed into one.
