# src/xrGame/ai/monsters/corpse_cover.h

> Declares the corpse-hiding cover policy, implemented in
> [`corpse_cover.cpp`](corpse_cover.cpp.md).

**Needs** — [`../../cover_evaluators.h`](../../cover_evaluators.h.md) · [`corpse_cover.cpp`](corpse_cover.cpp.md)
**Used by** — [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_path.cpp`](basemonster/base_monster_path.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`corpse_cover.cpp`](corpse_cover.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the creature-side cover policy and the one way it is configured. It is a header-only
declaration plus a trivial re-arming call; the scoring rule that matters is in the implementation
twin.

## `CMonsterCorpseCoverEvaluator`

- **construct** — takes the movement-restriction owner the generic evaluator needs, so that
  candidate cells outside the creature's permitted region are never offered
- **setup(min_distance, max_distance)** — re-arm for one query: reset the generic evaluator's
  best-so-far bookkeeping and install the acceptance band. Must be called before every search, or
  the previous search's winner survives into this one.
- **evaluate_cover** — score one candidate; see the implementation twin
- **evaluate_smart_cover** — accepts nothing. Creatures cannot use authored human cover volumes.
