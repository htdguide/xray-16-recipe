# src/xrPhysics/PHJointDestroyInfo.h

> Declares the break criterion attached to a breakable joint: two squared thresholds, a live reading of the constraint force the solver is carrying, and the latch that fires once.

**Needs** — [`PHJointDestroyInfo.cpp`](PHJointDestroyInfo.cpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHJoint.cpp`](PHJoint.cpp.md) · [`PHJoint.h`](PHJoint.h.md) · [`PHJointDestroyInfo.cpp`](PHJointDestroyInfo.cpp.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`ShellHit.cpp`](ShellHit.cpp.md)
**Tier floor** — T1: it hands the solver a buffer that the solver writes the per-joint reaction into.

## Purpose

Declares the surface implemented in [`PHJointDestroyInfo.cpp`](PHJointDestroyInfo.cpp.md).

A joint is breakable when it carries one of these. The record is the only thing standing between a
door that opens and a door that comes off its hinges; it holds the two thresholds, the solver's
reaction readout, and the identity of the split the break will cause.

## State

```text
RECORD JointDestroyInfo
  joint_feedback        : reaction readout   # force and torque on each of the joint's two bodies,
                                             # written by the solver every step
  sq_break_force        : real               # stored SQUARED, to avoid a root per test
  sq_break_torque       : real               # stored squared; see the notes in the implementation
  end_element           : int
  end_joint             : int                # where the shell splits when this joint breaks
  breaked               : bool               # a latch: set once, never cleared
```

**Invariants** — the thresholds are stored squared and compared against squared magnitudes. `breaked`
is a latch: once a joint has broken, the splitter will act on it at the next safe point, and
re-testing it in the meantime must not un-break it.

## Exported units

- **construct(break_force, break_torque)** — squares and stores both thresholds, zeroes the reaction
  readout, clears the latch.
- **`JointFeedback`** — hands out the buffer the dynamics library writes the joint's reaction into.
  This is the seam: the solver must be able to report, per joint per step, the force and torque it
  applied to hold that constraint.
- **`Breaked`** — reads the latch.
- **`Update`** — tests the reaction against the thresholds; contract in the implementation twin.

## Notes

`end_element` and `end_joint` are filled by the shell builder, not here: they record *which
contiguous range* of the shell's elements and joints separates when this joint goes. The range form
is the same one breakable elements use for shapes, and it works for the same reason — the shell
builder adds elements and joints in bone-hierarchy order, so a skeleton subtree is always a
contiguous range. See [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md).
