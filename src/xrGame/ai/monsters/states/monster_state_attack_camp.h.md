# src/xrGame/ai/monsters/states/monster_state_attack_camp.h

> Declares the ambush: take a cover point away from the enemy, watch the open ground, and occasionally creep out to where the enemy was last seen.

**Needs** — [`monster_state_attack_camp_inline.h`](monster_state_attack_camp_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`monster_state_attack_camp_inline.h`](monster_state_attack_camp_inline.h.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md)
**Tier floor** — T3: a three-phase cycle over a reserved cover point

## Purpose

Declares the surface implemented in
[`monster_state_attack_camp_inline.h`](monster_state_attack_camp_inline.h.md).

## State

```text
RECORD AttackCampState
  target_node : int     # the reserved cover vertex; locked against squadmates while camping
```

**Invariants** — `target_node` is chosen in `check_start_conditions` and consumed by
`initialize`, which is an unusual coupling: **the admission test has the side effect of picking
the destination.** It works because the only path into the state runs the test first, but it
means the two cannot be reordered and a rebuild should return the chosen cover from the test
rather than leaving it in a field.

The minimum engagement distance is a compiled-in twenty world units.

## Exported units

- construction — registers the three phases.
- `initialize` / `finalize` / `critical_finalize` — reserve and release the cover point.
- `check_start_conditions` — the admission test, which also picks the cover.
- `check_completion` — the three ways an ambush ends.
- `reselect_state` — the phase cycle.
- `setup_substates` — parameterise the approach and the watch.
- `check_force_state` — overridden to do nothing, suppressing the inherited preemption hook.
