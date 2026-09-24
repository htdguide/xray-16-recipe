# src/xrPhysics/PHFracture.h

> Declares a breakable seam inside one rigid body: the contiguous range of shapes, elements and joints that separates, the mass on each side of the cut, and the thresholds that decide.

**Needs** — [`PHFracture.cpp`](PHFracture.cpp.md) · [`PHImpact.h`](PHImpact.h.md) · [`PHDefs.h`](PHDefs.h.md) · [`PHElement.h`](PHElement.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHElement.cpp`](PHElement.cpp.md) · [`PHElement.h`](PHElement.h.md) · [`PHFracture.cpp`](PHFracture.cpp.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`PHShellSplitter.h`](PHShellSplitter.h.md) · [`ShellHit.cpp`](ShellHit.cpp.md)
**Tier floor** — T1: mass distributions are held in the dynamics library's own layout so they can be added and subtracted by it.

## Purpose

Declares the surface implemented in [`PHFracture.cpp`](PHFracture.cpp.md).

This is the *third* of the chapter's three breaking mechanisms, and the one that is easy to confuse
with the other two. A breakable **joint** ([`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md)) breaks
a constraint the solver already knows about. A **shell split**
([`PHShellSplitter.h`](PHShellSplitter.h.md)) is the bookkeeping that carries either kind of break
through to a new object. A **fracture** is different from both: it breaks a body *that has no joint
at the break*, along a seam that exists only as authored data, and the stress that breaks it must
therefore be *computed* rather than read from the solver.

## the range representation

The one idea everything here rests on:

```text
RECORD ShellSplitInfo               # what separates when this seam gives
  start_el_num,   end_el_num   : int     # a contiguous range of the shell's elements
  start_jt_num,   end_jt_num   : int     # a contiguous range of the shell's joints
  start_geom_num, end_geom_num : int     # a contiguous range of the element's shapes
  bone_id                      : int     # the skeleton bone the new element will own
```

**Invariants** — every one of these is a *half-open contiguous range*, and that is only sound
because the shell builder adds elements, joints and shapes in skeleton-hierarchy order, so that any
subtree of the skeleton is a contiguous run. A rebuild that stores them unordered must replace every
range with an explicit set and rewrite this file, [`PHElement.cpp`](PHElement.cpp.md)'s shape
passing and the whole of [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md).

`u16(-1)` in an end field means "not yet closed" — the seam has a start but the builder has not
reached its end. The mass distribution code branches on this.

**`sub_diapasone`** — when one range is removed from a list, every other range that contained or
followed it must shrink. This is that adjustment, and it is the operation performed over and over
whenever anything splits.

```text
FUNCTION subtract_range(INOUT from1, INOUT to1, from0, to0)
  IF either range is empty, or to1 lies at or before from0, or to1 is "not yet closed" THEN RETURN
  REQUIRE from0 >= from1 AND to0 <= to1          # the removed range must be nested inside this one
  to1 := to1 - (to0 - from0)
```

**Notes** — only the *end* moves. The removed range is required to be nested inside this one, so the
start is unaffected; the assertion is what enforces that nesting, and it is the invariant the whole
scheme depends on. Breaks are always nested or disjoint because the skeleton is a tree.

## `Fracture`

```text
RECORD Fracture EXTENDS ShellSplitInfo
  breaked        : bool           # latch
  first_mass     : mass distribution    # the side that stays with the element
  second_mass    : mass distribution    # the side that leaves
  break_force    : real           # threshold, then overwritten with the recorded blow
  break_torque   : real
  pos_in_element : vector
  add_torque_z   : real
```

**Invariants** — `first_mass + second_mass` equals the element's own mass, always. Every mass
operation on a fracture exists to maintain that.

**Notes on the overloading of four fields** — after a break latches, `break_force`, `break_torque`,
`pos_in_element` and `add_torque_z` stop being thresholds and become a *recording of the blow that
broke it*: the force vector goes into `pos_in_element` and the three torque components into the
other three. The recording is then never read — the code that would have applied it to the new
fragment is present but disabled. A rebuild should keep the thresholds immutable and drop the
recording; nothing observes it. This is the single most confusing thing in the file and it is
confusing in the original too.

**Exported units** — `Update` (the break test, contract in the implementation twin), `Breaked`, and
the mass arithmetic: `SetMassParts`, `MassSetZerro`, `MassAddToFirst`/`Second`,
`MassSubFromFirst`/`Second`, `MassSetFirst`/`Second`, `MassFirst`/`MassSecond`,
`MassUnsplitFromFirstToSecond` (move mass across the seam without changing the total).

## `FracturesHolder`

**Contract** — the set of seams belonging to one element, plus the two things a break test needs that
the seams do not own individually: the blows landed on the element this step, and reaction buffers
for the joints the solver would not otherwise report on.

```text
RECORD FracturesHolder            # owned by one Element
  has_breaks : bool               # latch: at least one seam gave
  fractures  : list<Fracture>     # ordered by shape range, innermost last
  impacts    : ImpactStorage      # this step's blows; cleared when no seam gave
  feedbacks  : list<reaction readout>   # reserved for non-contact joints
```

**Exported units**

- **`AddFracture`** / **`Fracture(index)`** / **`LastFracture`** — build and address the seam list.
- **`DistributeAdditionalMass`** — fold a newly added shape's mass into the correct side of every
  seam.
- **`SubFractureMass`** — remove a departing seam's mass from every other seam.
- **`AddImpact`** / **`Impacts`** — record and read this step's blows.
- **`PhTune`** — pre-solve: make sure every joint touching this body will report its reaction.
- **`PhDataUpdate`** — post-solve: run every seam's break test; returns whether any gave.
- **`SplitProcess`** — turn the broken seams into new elements.
- **`ApplyImpactsToElement`** — replay the recorded blows onto a freshly created fragment.
- **`CheckFractured`**, **`SplitFromEnd`**, **`InitNewElement`**, **`PassEndFractures`** — the split
  algorithm's steps; described in [`PHFracture.cpp`](PHFracture.cpp.md).
