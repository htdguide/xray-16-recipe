# src/xrPhysics/PHShellSplitter.h

> Declares the one place that watches every break criterion in a shell and, when one fires, carves the shell into two live objects mid-simulation.

**Needs** — [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`PHFracture.h`](PHFracture.h.md) · [`PHUpdateObject.h`](PHUpdateObject.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHDefs.h`](PHDefs.h.md) · [`Geometry.h`](Geometry.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHShell.cpp`](PHShell.cpp.md) · [`PHShell.h`](PHShell.h.md) · [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md) · [`ShellHit.cpp`](ShellHit.cpp.md)
**Tier floor** — T2: it is bookkeeping over index ranges; the solver contact is in the pieces it drives.

## Purpose

Declares the surface implemented in [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md).

Two things can break inside a shell — a joint between two bodies, and a seam inside one body — and
they are tested by different machinery ([`PHJointDestroyInfo.h`](PHJointDestroyInfo.h.md) and
[`PHFracture.h`](PHFracture.h.md)). What they have in common is what happens *afterwards*: a range
of the shell's elements and joints stops belonging to it and starts belonging to a new shell. This
file owns that common part, and it exists because the alternative — each break kind carving up the
shell itself — would duplicate the hardest code in the chapter twice.

## `Splitter`

```text
RECORD Splitter                       # "here is a place this shell can come apart"
  type    : one of {element, joint}
  element : int                       # index into the shell's element list
  joint   : int                       # index into the shell's joint list
  breaked : bool                      # latch, cleared after the split is performed
```

**Invariants** — the splitter list is in the same order as the elements and joints it refers to, which
is skeleton order. Every index-rewriting rule in the implementation depends on it. A splitter added
for an element is *inserted* at a recorded position rather than appended, to keep that order.

**Notes** — a splitter carries no thresholds and does no testing of its own. It is a *pointer to a
place that can break*, and the break criterion lives in the element's seams or the joint's destroy
info. One splitter per element and one per joint, no more — the shell's builder checks before adding.

## `SplitterHolder`

**Contract** — one per breakable shell. It is registered with the world as a per-step update object,
so its pre- and post-solve hooks run in the step whether or not the shell is otherwise busy.

```text
RECORD SplitterHolder
  shell          : Shell
  splitters      : list<Splitter>
  geom_root_map  : map<int, Shape>    # bone identifier → the shape a break is anchored at
  has_breaks     : bool               # any splitter latched this step
  unbreakable    : bool               # testing suspended
```

**Invariants** — `unbreakable` suspends the *testing*, not the seams: a shell can be made
unbreakable and breakable again without losing where it would have broken. Making it unbreakable
unregisters it from the step entirely, so a blocked breakable costs nothing.

## Exported units

**Step participation** — `PhTune` (pre-solve: make sure every seam's element will get its joint
reactions reported) and `PhDataUpdate` (post-solve: run every break test and accumulate the
latches). Both are the world's hooks, not the shell's.

**Breaking** — `SplitProcess` produces the new shells; `Breaked` reports whether anything latched;
`CheckSplitter`, `SplitJoint`, `SplitElement`, `ElementSingleSplit`, `PassEndSplitters` and
`InitNewShell` are its steps, described in [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md).

**Registration** — `AddSplitter` in two forms (append, or insert at a position), `isEmpty`,
`Activate`, `Deactivate`, `SetBreakable`, `SetUnbreakable`, `IsUnbreakable`.

**Lookup** — `AddToGeomMap` and `FindRootGeom`: bone identifier to the shape index that bone's break
is anchored at. Filled during the shell's build walk, and read when a hit must be attributed to a
breakable part.

## Notes

The holder is a *world update object* rather than something the shell calls, and that is deliberate:
its pre-solve hook must run before the solve of the step whose reactions it will read, and its
post-solve hook immediately after, with no chance of a caller forgetting. See
[`PHUpdateObject.h`](PHUpdateObject.h.md).

The result of a split is expressed as pairs of (new shell, the bone identifier its root owns). The
bone identifier is what the game layer needs to create a matching game object for the fragment — it
tells the object which part of the original model it is. The physics is done when the shells exist;
making them visible is [`xrGame`](../xrGame/README.md)'s problem.
