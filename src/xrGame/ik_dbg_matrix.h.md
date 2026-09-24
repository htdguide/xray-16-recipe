# src/xrGame/ik_dbg_matrix.h

> Declares the snapshot of every intermediate transform in a foot-placement solve, for offline inspection.

**Needs** — [`ik_calculate_data.h`](ik_calculate_data.h.md)
**Used by** — [`ik_dbg_matrix.cpp`](ik_dbg_matrix.cpp.md)
**Tier floor** — T3: a debugging record

## Purpose

Declares the record captured by [`ik_dbg_matrix.cpp`](ik_dbg_matrix.cpp.md). Not part of the
behaviour: it exists because a foot-placement bug is invisible in a single frame and is
diagnosed by comparing a run of frames.

A rebuild may omit this file entirely. It is described because the *set of transforms someone
found it necessary to record* is a good statement of what a foot-placement solve actually
consists of.

## State

```text
RECORD SolveSnapshot
  # the goal placement, for each of the limb's two end bones, in world and in object space
  b2_goal_world, b3_goal_world   : matrix
  b2_goal_local, b3_goal_local   : matrix
  b3_to_b3_goal                  : matrix   # the relation between the two, at the goal

  # the same four plus one, for the placement the limb STARTED the frame at
  b2_start_world, b3_start_world : matrix
  b2_start_local, b3_start_local : matrix
  b3_to_b3_start                 : matrix

  object_begin, object_end       : matrix   # the owner's transform before and after
  goal, goal_raw                 : matrix   # the goal as the solver received it, and as
                                            # the limb resolved it
  ref_bone                       : int (16-bit)

RECORD SnapshotHistory
  current : SolveSnapshot
  past    : list<SolveSnapshot>   # a bounded ring
```

**Notes** — that the owner's transform is recorded both before and after the solve is the
telling detail: a foot-placement bug is usually a disagreement between the frame the goal was
computed in and the frame it was applied in.
