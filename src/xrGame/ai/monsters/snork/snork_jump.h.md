# src/xrGame/ai/monsters/snork/snork_jump.h

> A snork-specific leap controller that was abandoned: the fields survive, every method is commented out, and nothing constructs it.

**Needs** — [`snork_jump.cpp`](snork_jump.cpp.md)
**Used by** — [`snork_jump.cpp`](snork_jump.cpp.md)
**Tier floor** — T3: dead code; nothing to implement

## Purpose

This is not a working part of the system. The type declares four fields and no callable
members — every method declaration is commented out, as is the entire body in
[`snork_jump.cpp`](snork_jump.cpp.md). Nothing includes it except its own implementation file,
nothing constructs it, and the snork's leap is instead handled by the shared motion-control
machinery configured in [`snork.cpp`](snork.cpp.md).

It is listed here because the mirror must be complete. A rebuild deletes both files.

## What the surviving fields imply

```text
RECORD AbandonedSnorkJump
  owner         : reference to the snork
  current_dist  : real            # last measured distance to an obstacle
  specific_jump : bool            # the distinguishing flag: a normal pounce, or the other kind
  target_object : reference       # what is being leapt at
  velocity_mask : int             # which velocity profiles the leap may use
```

The `specific_jump` flag is the one piece of design information left: the type was to
distinguish a straight pounce at an enemy from a second, different manoeuvre chosen when the
enemy was not in front of the snork. What that manoeuvre was meant to be is only visible in the
commented-out body; see [`snork_jump.cpp`](snork_jump.cpp.md).
