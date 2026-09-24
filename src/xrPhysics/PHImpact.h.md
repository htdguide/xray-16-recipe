# src/xrPhysics/PHImpact.h

> One recorded blow against a breakable object: a force, where it landed, and which shape it landed on — kept so it can be replayed onto the fragment that the blow separated.

**Needs** — _(none beyond the math vocabulary)_
**Used by** — [`TeleWhirlwind.cpp`](../xrGame/TeleWhirlwind.cpp.md) · [`TeleWhirlwind.h`](../xrGame/TeleWhirlwind.h.md) · [`PHFracture.h`](PHFracture.h.md)
**Tier floor** — T3: a three-field record.

## Purpose

When a breakable object is hit, the hit does two things at different times: it pushes the whole body
*now*, and — if it turns out to break the object — it must push the *fragment* that comes off, on a
later step, after the fragment exists. That second use needs the blow remembered, which is all this
record is.

## State

```text
RECORD Impact
  force : vector    # already scaled to a per-step force, not an impulse
  point : vector    # in the body's frame at the moment of the hit
  geom  : int       # index of the shape that was hit, within the element

LIST ImpactStorage = list<Impact>
```

**Invariants** — `geom` indexes the element's shape list at the moment the impact was recorded.
Shapes are renumbered when an element splits, so the storage is consumed or cleared on the same step
the split happens; a stale index would attribute a later blow to the wrong half. The storage is
filled from the element's bone-attributed impulse path and cleared in the post-solve pass whenever
no break occurred — see [`PHElement.cpp`](PHElement.cpp.md) and [`PHFracture.cpp`](PHFracture.cpp.md).
