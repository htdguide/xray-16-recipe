# src/xrGame/ai/monsters/melee_checker.h

> Declares the melee-range judge: when a creature may start swinging and when it must stop.

**Needs** — [`melee_checker.cpp`](melee_checker.cpp.md) · [`melee_checker_inline.h`](melee_checker_inline.h.md) · [Seam: Static collision database](../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`melee_checker.cpp`](melee_checker.cpp.md) · [`melee_checker_inline.h`](melee_checker_inline.h.md)
**Tier floor** — T2: holds a ray-query result buffer it reuses across calls to avoid per-call allocation in a per-tick path

## Purpose

Declares the surface implemented in [`melee_checker.cpp`](melee_checker.cpp.md) and
[`melee_checker_inline.h`](melee_checker_inline.h.md). One of these belongs to each creature
that attacks by contact; the creature's attack state asks it the two questions below every
tick.

## Exported units

- `bind` — attaches the judge to its creature. Must happen before any other call.
- `load` — reads the four authored distances from a configuration section.
- `begin_attack` — resets the adaptation for a fresh attack bout.
- `report_swing` — records whether one swing connected; this is what drives the adaptation.
- `distance_to_enemy` — the effective distance to a target, measured from the creature's head
  and corrected by a ray against the collision database.
- `min_distance` / `max_distance` — the current inner and outer edges of the melee window.
- `can_start_melee` / `should_stop_melee` — the two questions the attack state asks.

The record also holds a reusable ray-query result buffer, shared by every call, and a fixed
two-entry swing history.
