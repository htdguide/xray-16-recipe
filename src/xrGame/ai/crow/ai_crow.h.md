# src/xrGame/ai/crow/ai_crow.h

> Declares the crow: a flying ambient creature with four states, no perception and no navigation.

**Needs** — [`entity_alive.h`](../../entity_alive.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../../../Include/xrRender/KinematicsAnimated.h.md) · [`ai_crow.cpp`](ai_crow.cpp.md)
**Used by** — [`ai_crow.cpp`](ai_crow.cpp.md) · [`ai_crow_script.cpp`](../../ai_crow_script.cpp.md)
**Tier floor** — T2: declares a per-frame-updated entity with a fixed-size animation and sound table

## Purpose

Declares the surface implemented in [`ai_crow.cpp`](ai_crow.cpp.md). The declaration is
worth a glance on its own because of what it *refuses*: the crow derives from the plain
entity, not from the creature base, so it has no memory, no enemies, no navigation mesh
position and no planner. Everything a creature normally inherits, the crow opts out of.

## Exported units

- **the crow entity** — the creature itself; four states, a flight integrator, a death path.
- **state identifiers** — flying level, climbing, falling dead, landed dead.
- **animation group** — a small random-variant motion list; five groups (idle, fly, death,
  death idle, death landed), each holding at most eight variants.
- **sound group** — the same shape for sounds; one group (idle call), at most eight variants.
- **hit-animation finished** — the callback the animation layer fires when the death fall
  animation ends.
- **the negative answers** — casts no shadow, receives no shadow, is not drawn in the
  first-person view, is not affected by anomalies, does not occupy a navigation mesh
  position, and supplies no weapon geometry. Each of these is a deliberate exclusion, and
  together they are most of what makes a crow cheap.
- **perception constants** — a 150-degree field of view and a 30-unit range are declared,
  fixed in code rather than read from configuration, and never consulted, because the crow
  has no perception.
