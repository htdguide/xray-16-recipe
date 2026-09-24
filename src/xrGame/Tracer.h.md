# src/xrGame/Tracer.h

> Declares the bullet-in-flight renderer implemented in [`Tracer.cpp`](Tracer.cpp.md).

**Needs** — [`xrUICore/ui_defs.h`](../xrUICore/ui_defs.h.md)
**Used by** — [`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`Tracer.cpp`](Tracer.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CTracer`, which owns the tracer material and the colour table and draws one
bullet. Substance is in [`Tracer.cpp`](Tracer.cpp.md).

Exported units:

- `CTracer` — the renderer; constructed once, from configuration.
- `Render` — emits one round's geometry, given its head position, the midpoint of its
  streak, its direction, the streak's length and width, a colour identifier, its speed, and
  whether it is the local player's own round.

## Notes

The ballistics manager is declared a friend so it can reach the material and colour table
directly. That is a shortcut around the interface, not a design: the two are used together
and were not given a clean boundary. A rebuild should either fold this into the ballistics
manager or give it real accessors.
