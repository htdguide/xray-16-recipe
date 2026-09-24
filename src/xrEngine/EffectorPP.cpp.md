# src/xrEngine/EffectorPP.cpp

> The default post-process effector: count down, contribute nothing.

**Needs** — [`EffectorPP.h`](EffectorPP.h.md) · [`CameraManager.h`](CameraManager.h.md) · [`device.h`](device.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

The base of the post-process effector family. A camera-stacked effect that wants to tint,
blur, shake or otherwise colour the finished frame derives from this and overrides the one
per-frame call; what lives here is the default that every such effect inherits — a
lifetime that runs down on its own and a contribution of nothing.

It is a separate file only because the base must exist somewhere; a rebuild may fold it
into the effector interface it belongs to.


## `apply`

**Contract** — Called once per frame with an all-zero parameter set to fill in. The base implementation decrements the lifetime by the frame's scaled elapsed time and reports success without touching the parameters; every real effect overrides it.

```text
FUNCTION apply(out params) -> bool
  life_time = life_time - frame_delta_seconds
  RETURN true
```

**Notes** — It returns success unconditionally, even once the lifetime has gone negative, unlike its camera counterpart which reports expiry from the same place. The manager tests validity separately before calling, so the difference is invisible — but it means a subclass calling into the base cannot use the return value to learn that time has run out. A rebuild should make the two families agree.

**Notes** — The parameter set handed in starts at *zero*, not at the neutral value, because the manager composites contributions as deviations. An effector that wants no effect on a channel must leave that channel at zero, and an effector that wants to force a channel to neutral must write the neutral value explicitly.
