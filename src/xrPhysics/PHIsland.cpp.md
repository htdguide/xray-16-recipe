# src/xrPhysics/PHIsland.cpp

> Chooses the solver for one island each step, and scrubs non-finite body state.

**Needs** — [`PHIsland.h`](PHIsland.h.md) · [`Physics.h`](Physics.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`ph_valid_ode.h`](ph_valid_ode.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHIsland.h`](PHIsland.h.md)
**Tier floor** — T1: reads and writes the dynamics library's body records directly.

## Purpose

Three short operations that are load-bearing out of proportion to their length: which solver an
island gets, how a sleeping island is woken, and how a poisoned body is made safe.

## `step`

**Contract** — advances one island by the *fixed* timestep. Silently returns if this island has
been absorbed by a merge. Allocates nothing.

```text
FUNCTION step(requested_step)
  IF NOT is_live THEN RETURN               # the absorber steps this island's contents
  IF prefers_exact_integration AND joint_count < exact_integration_joint_limit THEN
    solve_exact(fixed_step)
  ELSE
    solve_iterative(fixed_step)
```

**Invariants** — the argument is ignored; the world's `fixed_step` is used unconditionally. That is
deliberate: the determinism requirement in
[§6 of the system requirements](../../SYSTEM-REQUIREMENTS.md#6-conformance) says the same inputs
must produce the same trajectories, and a variable timestep breaks that immediately. A rebuild must
keep the solver's step constant and let the outer loop vary the *number* of steps
(see [`PHWorld.cpp`](PHWorld.cpp.md)).

**Notes** — two solvers exist because they trade differently. The exact one factors the whole
constraint system and is accurate but superlinear in constraint count; the iterative one runs a
fixed number of relaxation sweeps and is linear but drifts, which shows up as a jointed structure
that sags or buzzes. The engine's rule: an object may *ask* for the exact solver
(`set_prefere_exact_integration`, used by ragdolls and vehicles where sag is visible), and gets it
only while its island stays under a small constraint count. Above that it silently falls back.
That threshold — a few tens of constraints — is where the exact solve stops fitting the frame
budget on hardware of the era; re-measure it rather than copying it.

## `enable`

**Contract** — clears the sleeping flag on every body in the live island. Used when the world must
guarantee that a touch-resolution pass actually moves things.

## `repair`

**Contract** — walks the island's bodies and replaces any non-finite angular velocity, linear
velocity or position with zero, and any non-finite orientation with the identity. Idempotent; costs
one pass over the island.

**Notes** — see [`PHIsland.h`](PHIsland.h.md) for why this exists at all. The choice of *zero* for
a bad position is not a recovery, it is a containment: the object teleports to the frame origin and
the subsequent out-of-bounds check in [`PHShell.cpp`](PHShell.cpp.md) disables it. A rebuild could
do better by restoring the last known-good state — the machinery for that already exists in
[`PHValideValues.h`](PHValideValues.h.md) — and it is not clear why this path does not use it.
