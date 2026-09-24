# src/xrGame/danger_explosive.h

> Declares the tracked-grenade record, implemented in [`danger_explosive.cpp`](danger_explosive.cpp.md) and [`danger_explosive_inline.h`](danger_explosive_inline.h.md).

**Needs** — [`GameObject.h`](GameObject.h.md) · [`Explosive.h`](Explosive.h.md) · [`danger_explosive_inline.h`](danger_explosive_inline.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`agent_explosive_manager.h`](agent_explosive_manager.h.md) · [`danger_explosive.cpp`](danger_explosive.cpp.md) · [`danger_explosive_inline.h`](danger_explosive_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the record a creature keeps for one grenade it is reacting to. Substance is in
[`danger_explosive.cpp`](danger_explosive.cpp.md).

Its fields are public and there are no accessors — the record is a plain aggregate that its
owner reads and writes directly. That is the right shape: it is one creature's private
scratch note, not a shared object with an interface.

Exported units:

- `CDangerExplosive` — the record: the explosive, the same entity as a game object, the
  creature reacting, and the time it was noticed.
- Comparison against an explosive reference — identity, for callers holding the object.
- Comparison against an entity identifier — for callers holding only the number, which is
  how a grenade reported by a squadmate arrives.
