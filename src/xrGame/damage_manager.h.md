# src/xrGame/damage_manager.h

> Declares the per-bone damage multiplier mix-in, implemented in [`damage_manager.cpp`](damage_manager.cpp.md).

**Needs** — _(none beyond the engine's object and configuration types)_
**Used by** — [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md) · [`DestroyablePhysicsObject.h`](DestroyablePhysicsObject.h.md) · [`Entity.cpp`](Entity.cpp.md) · [`Entity.h`](Entity.h.md) · [`damage_manager.cpp`](damage_manager.cpp.md) · [`entity_alive.cpp`](entity_alive.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the mix-in that gives a creature per-bone hit and wound multipliers. Substance is in
[`damage_manager.cpp`](damage_manager.cpp.md).

Its own load-bearing content is that the class holds only **two** numbers — the section-wide
defaults — while the per-bone table lives in the carrier's skeleton. A reader looking for the
damage table in this class will not find it.

Exported units:

- `CDamageManager` — the mix-in: two default factors and the carrier.
- `reload` — read a damage table, in two spellings: the section that is the table, or a
  section and a key naming one.
- `HitScale` — the query: struck bone plus an aimed-first-bullet flag, yielding the hit and
  wound multipliers.
- `_construct` — resolve the carrier; part of the engine's two-phase object construction.
- `init_bones` / `load_section` — private: seed every bone with the defaults, then overlay
  the authored lines.
