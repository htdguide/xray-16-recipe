# src/xrGame/DelayedActionFuse.h

> Declares the condition-coupled fuse implemented in [`DelayedActionFuse.cpp`](DelayedActionFuse.cpp.md).

**Needs** — [`DelayedActionFuse.cpp`](DelayedActionFuse.cpp.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`DelayedActionFuse.cpp`](DelayedActionFuse.cpp.md) · [`ExplosiveItem.cpp`](ExplosiveItem.cpp.md) · [`ExplosiveItem.h`](ExplosiveItem.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the fuse mixin: three state flags, a time field and a rate field whose meanings
change on arming, and the two operations it demands of its host. Substance in
[`DelayedActionFuse.cpp`](DelayedActionFuse.cpp.md).

Exported units:

- `Initialize` — set duration and arming threshold.
- `CheckCondition` — arm if the host's condition has fallen to the threshold; answers
  whether it did.
- `SetTimer` — arm unconditionally, snapping the condition to the threshold.
- `Update` — advance one frame, drain condition against the schedule, answer whether the
  fuse fired.
- `Time` — remaining duration, meaningful in both the armed and unarmed states.
- `isActive`, `isInitialized` — the two state predicates.
- `ChangeCondition`, `StartTimerEffects` — what the fuse demands of its host: apply a
  condition delta, and begin the visible or audible telegraph of an armed fuse.
