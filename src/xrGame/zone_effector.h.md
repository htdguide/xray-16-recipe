# src/xrGame/zone_effector.h

> Declares the anomaly's screen effect: a named post-process, two radius fractions, and the strength the camera reads back each frame.

**Needs** — [`zone_effector.cpp`](zone_effector.cpp.md) · [`xrServerEntities/alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`CustomZone.cpp`](CustomZone.cpp.md) · [`zone_effector.cpp`](zone_effector.cpp.md)
**Tier floor** — T2: holds a reference into the camera's effect stack

## Purpose

Declares the surface implemented in [`zone_effector.cpp`](zone_effector.cpp.md). A zone
that wants a screen effect owns one of these by value and drives it.

Exported units:

- `Load(section)` — read the effect name and the two radius fractions.
- `Update(distance, radius, hit_type)` — attach, detach, and recompute strength.
- `Stop` — detach; also called from the destructor.
- `GetFactor` — the strength callback the camera's effect stack reads.
- `m_pActor` — the attached player, exposed directly so the owning zone can ask whether the
  effect is currently on the player.
- `Activate` — private; attaching is driven entirely by `Update`.

## State

See [`zone_effector.cpp`](zone_effector.cpp.md).

**Notes** — The damage type is part of `Update`'s signature rather than loaded configuration,
which says the *same* effector can be driven with different damage types over its life. In
practice a zone always passes its own, but the coupling is deliberately at the call site: the
effector knows how to ask an outfit for protection, and not which kind of harm the zone does.
