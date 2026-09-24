# src/xrPhysics/phvalide.cpp

> Answers whether a position is inside the level, and produces the diagnostic when it is not.

**Needs** — [`phvalide.h`](phvalide.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md)
**Used by** — [`phvalide.h`](phvalide.h.md)
**Tier floor** — T2: a box containment test plus string formatting.

## Purpose

Physics can send an object anywhere, including to coordinates where single-precision floats stop
being useful. The level's own bounding box, widened at load, is the fence: an object outside it is
not recoverable and is disabled rather than simulated. This file holds the test and the message.

## State

```text
level_bounds : box      # owned elsewhere (set when a level loads); read-only here
```

**Invariants** — the bounds must be set before any physics object is activated. A level whose bounds
are still empty rejects every position, which shows up as everything disabling on the first step —
a distinctive and useful failure mode rather than a subtle one.

## `valid_pos`

**Contract** — true when the point lies inside the current level's bounds. No side effects, no
allocation, safe to call from any thread once the level is loaded.

```text
FUNCTION valid_pos(p) -> bool
  RETURN level_bounds.contains(p)
```

## `ph_boundaries`

**Contract** — returns the current level's bounds. Exported so that callers outside the module —
notably the code that decides where a thrown or dropped object may come to rest — test against the
same box rather than keeping their own copy.

## `dbg_valid_pos_string`

**Contract** — formats the failure: the offending position, the bounds, and a full textual dump of
the object that owns it, obtained through the object's own dump entry point
([`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md)). Allocates a string; only ever called on a
failing path, so its cost does not matter. Tolerates a missing object and omits that section.

**Notes** — the dump is the whole value of this function. A position going out of bounds is almost
never interesting by itself; what is interesting is *which* object, with which visual, in which
state, under which shell. The object dump answers that in one step instead of a debugging session.

A rebuild should keep the structure — predicate, bounds accessor, and a formatter that reaches the
object — and is free to make the diagnostic always available rather than development-only. The
original compiles it out because the dump pulls the whole object hierarchy into the physics module's
link, which is a C++ concern and not a reason.
