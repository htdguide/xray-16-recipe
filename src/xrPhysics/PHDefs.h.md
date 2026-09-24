# src/xrPhysics/PHDefs.h

> The collection type names the shell machinery passes around.

**Needs** — _(none beyond the core containers)_
**Used by** — [`PHSkeleton.h`](../xrGame/PHSkeleton.h.md) · [`PHElement.h`](PHElement.h.md) · [`PHFracture.h`](PHFracture.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHShellSplitter.h`](PHShellSplitter.h.md) · [`PhysicsShell.h`](PhysicsShell.h.md)
**Tier floor** — T3: naming, no decisions.

## Purpose

C++ forces a shared header for the container aliases used across
[`PHShell`](PHShell.cpp.md), [`PHElement`](PHElement.cpp.md) and the splitter. Almost
nothing here survives into a rebuild as itself.

## State

The two that carry a decision:

```text
RECORD shell_root                 # names a sub-tree of a shell
  shell : PhysicsShell
  root  : int (16-bit)            # index of the element that roots it

# a shell's elements are stored in a list whose ORDER IS PART OF THE CONTRACT:
#   index 0 is the root element, and element index is the handle used by
#   save games and by the network protocol.  Reordering the list changes
#   the meaning of a saved or transmitted shell.
ELEMENT_STORAGE = list<PhysicsElement>
JOINT_STORAGE   = list<PhysicsJoint>
```

**Notes** — the reverse iterators aliased here are not incidental: shell tear-down and the
splitter both walk elements from the leaves back to the root, because a child must be
detached before its parent is destroyed. A rebuild keeps that ordering requirement and drops
the aliases.

The `shell_root` pair is the splitter's currency — when a shell breaks apart
([`PHShellSplitter.cpp`](PHShellSplitter.cpp.md)), each new fragment is described by the
shell it came from plus the element that becomes its root.
