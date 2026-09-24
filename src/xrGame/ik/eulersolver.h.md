# src/xrGame/ik/eulersolver.h

> Declares the four Euler conventions the solver admits, the decomposition of a rotation
> into three joint angles under one of them, and the same decomposition expressed as a
> function of the swivel angle.

**Needs** — [`math3d.h`](math3d.h.md) · [`jtlimits.h`](jtlimits.h.md)
**Used by** — [`eulersolver.cxx`](eulersolver.cxx.md) · [`limb.cxx`](limb.cxx.md) · [`limb.h`](limb.h.md)
**Tier floor** — T2. A small record holding three joint descriptions.

## Purpose

Declares the surface implemented in [`eulersolver.cxx`](eulersolver.cxx.md), which carries
the substance.

## Exported units

**An enumeration of four rotation conventions.** Each names three axes in composition
order, with case carrying the sign of the rotation — so one entry means "rotate about
*z*, then about *x*, then about *y*", and another means the same axes with the first two
negated. The four are, in the vendor's words, the conventions for a left shoulder or hip
or ankle, a left wrist, a right wrist, and a right shoulder or hip or ankle. **The values
index a table and must not be renumbered.** The shipped engine uses only the first, for
both spherical joints of a leg — the right-side variants exist for a mirrored skeleton the
engine does not build.

- Decompose a rotation matrix into three angles under a named convention, choosing one of
  the two solution families.
- The same, returning both families at once.
- The inverse: compose three angles back into a rotation matrix.

**A swivel-parameterized decomposer**, built once per goal from three matrices that
express the rotation as `cos ψ · C + sin ψ · S + O`, plus the three joints' authored
limits. Its operations: the three joint angles for a given swivel angle and family, their
derivatives, the singular swivel angles, and — the reason it exists — **the set of swivel
angles for which all three joints stay inside their limits**, one set per family.
