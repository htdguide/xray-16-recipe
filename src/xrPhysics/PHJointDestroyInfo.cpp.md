# src/xrPhysics/PHJointDestroyInfo.cpp

> Decides, once per step, whether the force the solver is spending to hold a joint together has exceeded what that joint can bear.

**Needs** — [`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md)
**Tier floor** — T1: it reads a reaction buffer the dynamics library filled in its own layout.

## Purpose

The break criterion for joints, which is the simplest of the three breaking mechanisms in this
chapter (the others being the per-element fracture in [`PHFracture.cpp`](PHFracture.cpp.md) and the
shell split in [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md)). Its whole idea is that a
constraint solver already knows how hard it is working: the reaction it applies to keep two bodies
joined *is* the stress in the joint, and no extra physics is needed to measure it.

## `construct`

**Contract** — takes a break force and a break torque in the authored units of the game data, squares
both, zeroes the four reaction vectors, and clears the break latch. Allocates nothing.

**Notes** — the thresholds arrive from the model's authored bone data, expressed in the original
dynamics library's force units. This is the part that transfers least cleanly to a different
library: a rebuild swapping the solver must rescale every breakable joint in the shipped models, or
calibrate one global factor against a known case. The
[rigid-body seam](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) says as much.

## `Update`

**Contract** — tests the solver's reaction on both bodies against the force threshold and latches the
break if any component exceeds it. Returns whether the break fired *on this call*; the latch itself
is read separately. Reads a console-tunable global scale. No side effects beyond the latch.

```text
FUNCTION update() -> bool
  threshold := sq_break_force / global_break_factor      # a player-facing difficulty knob
  FOR EACH r IN (force_on_body_1, force_on_body_2, torque_on_body_1, torque_on_body_2)
    IF squared_magnitude(r) > threshold THEN
      breaked := true
      RETURN true
  RETURN false
```

**Invariants** — the latch is set-only. A joint that broke stays broken until the shell splitter
removes it.

**Notes** — **the torque threshold is never used.** All four tests — including the two torque
readings — compare against the *force* threshold. `sq_break_torque` is stored and never read. This
is either a long-standing bug or a deliberate simplification that survived because the authored
torque values were tuned around it; the shipped game data is balanced against the behaviour as
written, so a rebuild that "fixes" it by testing torque against the torque threshold will find
doors and limbs breaking at different moments than the original. Reproduce the behaviour, and treat
the torque field as unused.

The global factor divides the threshold rather than multiplying the reading, so a larger factor
makes everything break *more* easily. It is exposed as a console variable so the difficulty of
tearing a level apart is tunable without re-authoring the models.
