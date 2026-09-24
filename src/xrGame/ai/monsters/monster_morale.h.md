# src/xrGame/ai/monsters/monster_morale.h

> Declares morale: one scalar in the unit interval that rises or falls at a state-dependent rate and decides whether a creature is willing to fight.

**Needs** — [`monster_morale.cpp`](monster_morale.cpp.md) · [`monster_morale_inline.h`](monster_morale_inline.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`monster_morale.cpp`](monster_morale.cpp.md) · [`monster_morale_inline.h`](monster_morale_inline.h.md)
**Tier floor** — T3: one scalar and a three-valued mode

## Purpose

Declares the surface implemented in [`monster_morale.cpp`](monster_morale.cpp.md) and
[`monster_morale_inline.h`](monster_morale_inline.h.md).

## Exported units

- `bind`, `load(section)`, `reinit` — attachment, the six authored numbers, and the reset to
  full morale in the stable mode.
- `on_hit` / `on_attack_success` — the two discrete nudges.
- `update_schedule(elapsed)` — the continuous drift.
- `set_despondent`, `set_taking_heart`, `set_stable` — set the drift mode.
- `is_despondent` — the question the brains ask.
- `morale` — the raw value.
