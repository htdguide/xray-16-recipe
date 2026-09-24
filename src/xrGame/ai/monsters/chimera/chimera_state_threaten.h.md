# src/xrGame/ai/monsters/chimera/chimera_state_threaten.h

> Declares the chimera's intimidation behaviour: roar, stalk closer, roar again — used on anything that is not yet a sworn enemy.

**Needs** — [`state.h`](../state.h.md) · [`chimera_state_threaten_inline.h`](chimera_state_threaten_inline.h.md)
**Used by** — [`chimera_state_threaten_inline.h`](chimera_state_threaten_inline.h.md)
**Tier floor** — T3: selection over shared movement states

## Purpose

Declares the surface implemented in [`chimera_state_threaten_inline.h`](chimera_state_threaten_inline.h.md). Nothing in the engine instantiates this state — see that twin's Notes.

## `ChimeraThreatenState`

A composite state with four declared substate slots — walk, face enemy, roar, stalk — of which three are ever registered. It holds one field, the time the last threat display ended, and the three constants that gate the behaviour:

```text
min_distance_to_enemy = 3 world units    # closer than this, threatening is over
morale_threshold      = 0.8              # declared, never read
threaten_cooldown     = 10000 ms         # between displays
```
