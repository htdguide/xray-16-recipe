# src/xrGame/script_action_planner_wrapper_inline.h

> Empty.

**Needs** — _(none)_
**Used by** — [`script_action_planner_wrapper.h`](script_action_planner_wrapper.h.md)
**Tier floor** — T4: nothing to implement.

## Purpose

The script planner adapter has no constructor and no inline members, so this file holds
nothing. It exists because every other adapter in the family has an inline companion and the
header includes one unconditionally.

A rebuild should not recreate this file. It is listed here only so the mirror is complete.
