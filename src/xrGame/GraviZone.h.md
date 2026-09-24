# src/xrGame/GraviZone.h

> Declares the gravitational anomaly implemented in [`GraviZone.cpp`](GraviZone.cpp.md).

**Needs** — [`CustomZone.h`](CustomZone.h.md) · [`ai/monsters/telekinesis.h`](ai/monsters/telekinesis.h.md)
**Used by** — [`GraviZone.cpp`](GraviZone.cpp.md) · [`Mincer.cpp`](Mincer.cpp.md) · [`Mincer.h`](Mincer.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the two-layer gravitational anomaly as an abstract base plus a concrete leaf.
The base holds the configuration and the field rules and demands a telekinesis controller
from its implementor; the leaf supplies one. Substance is in
[`GraviZone.cpp`](GraviZone.cpp.md).

Exported units:

- `CBaseGraviZone` — the pull-and-blowout field, the telekinesis lift cycle, and the
  configured strengths, radii and effect names.
- `Affect`, `AffectPull`, `AffectPullAlife`, `AffectPullDead`, `AffectThrow` — the
  per-victim effect, split by whether the victim is alive.
- `ThrowInCenter`, `CheckAffectField`, `BlowoutRadiusPercent` — the overridable geometry
  of the field, so a derived anomaly can move the focus or make the boundary depend on
  the victim.
- `BlowoutState`, `IdleState`, `shedule_Update` — the state ticks.
- `Telekinesis` — what the base demands of an implementor: one telekinesis controller,
  which may be owned or shared.
- `CGraviZone` — the concrete anomaly, owning its controller.
