# src/xrGame/ai/monsters/states/monster_state_attack_run.h

> Declares the approach: run at the enemy's navigation vertex, replanning on the creature's own schedule, until close enough to bite.

**Needs** — [`monster_state_attack_run_inline.h`](monster_state_attack_run_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`burer_state_attack_inline.h`](../burer/burer_state_attack_inline.h.md) · [`controller_state_attack_inline.h`](../controller/controller_state_attack_inline.h.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`monster_state_attack_run_inline.h`](monster_state_attack_run_inline.h.md)
**Tier floor** — T3: a leaf state issuing path requests

## Purpose

Declares the surface implemented in
[`monster_state_attack_run_inline.h`](monster_state_attack_run_inline.h.md).

## State

```text
RECORD AttackRunState
  path_rebuild_at : int    # initialised to zero at construction and never read or written again
```

**Notes** — the rebuild timestamp is vestigial. Route replanning is throttled by a value the
*creature* supplies to the path builder each tick, not by a timer held here. A rebuild deletes
the field.

## Exported units

- construction — takes the creature; registers no children.
- `initialize` — prepare the path builder.
- `execute` — one tick of the approach.
- `finalize` / `critical_finalize` — both turn route extrapolation back off.
- `check_start_conditions` / `check_completion` — the two distance thresholds.
- `remove_links` — forwards to the base cascade.
