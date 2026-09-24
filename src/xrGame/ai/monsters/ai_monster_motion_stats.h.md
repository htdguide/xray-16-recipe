# src/xrGame/ai/monsters/ai_monster_motion_stats.h

> Declares the short ring of recent positions a creature uses to notice it is not actually moving.

**Needs** — [`ai_monster_defs.h`](ai_monster_defs.h.md) · [`ai_monster_motion_stats.cpp`](ai_monster_motion_stats.cpp.md)
**Used by** — [`ai_monster_motion_stats.cpp`](ai_monster_motion_stats.cpp.md)
**Tier floor** — T3: a ten-entry ring of (speed, position, time)

## Purpose

Declares the surface implemented in
[`ai_monster_motion_stats.cpp`](ai_monster_motion_stats.cpp.md). The capacity — ten
samples — is fixed here and is the only number the declaration carries.

## Exported units

- **motion statistics** — holds a reference to its creature, records one sample per think,
  and answers whether the last few samples show real progress.
