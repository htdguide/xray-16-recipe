# src/xrGame/stalker_base_action.h

> Declares the base every stalker planner action derives from.

**Needs** — [`stalker_base_action.cpp`](stalker_base_action.cpp.md) · [`action_script_base.h`](action_script_base.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md) · [`stalker_alife_actions.h`](stalker_alife_actions.h.md) · [`stalker_alife_task_actions.cpp`](stalker_alife_task_actions.cpp.md) · [`stalker_alife_task_actions.h`](stalker_alife_task_actions.h.md) · [`stalker_anomaly_actions.h`](stalker_anomaly_actions.h.md) · [`stalker_base_action.cpp`](stalker_base_action.cpp.md) · [`stalker_combat_action_base.h`](stalker_combat_action_base.h.md) · [`stalker_death_actions.h`](stalker_death_actions.h.md)
**Tier floor** — T2: one object per action per creature, instantiated at brain setup.

## Purpose

Declares the surface implemented in [`stalker_base_action.cpp`](stalker_base_action.cpp.md).
Every leaf action in a stalker's brain — combat, danger, anomaly, death, smart cover —
inherits from here, which is what guarantees the two housekeeping rules in the `.cpp` twin
run for *all* of them rather than being remembered action by action.

## Exported units

- construction from the creature plus a human-readable action name (the name exists only
  for the planner log).
- `initialize()` — action entry hook.
- `execute()` — per-cycle hook.
- `finalize()` — action exit hook.
- `object()` — the creature this action belongs to, as a reference rather than a handle:
  an action is never separated from its creature, so the nullable case is asserted away
  rather than handled.
