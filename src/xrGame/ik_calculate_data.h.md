# src/xrGame/ik_calculate_data.h

> The working set one limb's foot-placement solve is carried in: the inputs, the persistent state, and the outputs.

**Needs** — [`ik_calculate_state.h`](ik_calculate_state.h.md) · [`ik/IKLimb.h`](ik/IKLimb.h.md)
**Used by** — [`IKFoot.cpp`](IKFoot.cpp.md) · [`IKFoot.h`](IKFoot.h.md) · [`IKLimb.cpp`](ik/IKLimb.cpp.md) · [`IKLimb.h`](ik/IKLimb.h.md) · [`ik_calculate_data.cpp`](ik_calculate_data.cpp.md) · [`ik_dbg_matrix.cpp`](ik_dbg_matrix.cpp.md) · [`ik_dbg_matrix.h`](ik_dbg_matrix.h.md) · [`ik_limb_state.h`](ik_limb_state.h.md)
**Tier floor** — T2: a record bundling one solve's parameters

## Purpose

The foot-placement solve for one limb passes through a dozen functions — sampling the
animation, testing the ground, choosing a goal, blending toward it, applying the result — and
every one of them needs most of the same things. This record is that bundle. Its
implementation file, [`ik_calculate_data.cpp`](ik_calculate_data.cpp.md), holds only the
construction, so the shape is the substance and it is stated here.

## State

```text
RECORD CalculateData
  angles     : optional<list<real>>   # the solved joint angles; filled by the solver,
                                      # read by the application step. Not owned.
  limb       : Limb                   # which limb this solve is for
  object     : matrix                 # the owning object's world transform at solve time
  do_collide : bool                   # whether the ground test should run this frame
  state      : CalculateState         # the limb's persistent between-frame state
  cl_shift   : vector = (0,0,0)       # the correction collision asked for
  apply      : bool                   # whether the result should be written into the pose
  l          : real                   # blend fraction for position, this frame
  a          : real                   # blend fraction for orientation, this frame
```

**Invariants** — the object transform is captured **once** at the start of the solve and held
by reference for its duration. Every position in the solve is expressed against that one
snapshot, so that a solve is internally consistent even though the owner may be moving. A
rebuild that re-reads the owner's transform partway through will produce placements that
disagree with each other by a frame of motion.

The joint-angle array is borrowed, not owned: it belongs to the solver's scratch storage and
is valid only within one solve.

The two blend fractions are per-frame outputs of the blend-speed calculation, separate for
position and orientation for the same reason their speeds are separate.

The apply flag is decided late: a solve can complete and still be discarded, because the
result may be worse than leaving the animation alone.

## `goal`

**Contract** — the placement the solve is driving toward, written into a caller-supplied
transform and returned. A convenience over reaching into the persistent state, and the only
method on the record.

## Debug instrumentation

**Notes** — a compile-time switch records a history of every intermediate transform for
inspection; see [`ik_dbg_matrix.h`](ik_dbg_matrix.h.md). It is off in every shipped
configuration and is not part of the behaviour.
