# src/xrGame/smart_cover_planner_target_selector.h

> Declares the sub-planner that decides *which* smart-cover behaviour a creature in cover should currently be pursuing.

**Needs** — [`smart_cover_detail.h`](smart_cover_detail.h.md) · [`action_planner_action.h`](action_planner_action.h.md) · [`smart_cover_planner_target_selector_inline.h`](smart_cover_planner_target_selector_inline.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_planner_target_provider.h`](smart_cover_planner_target_provider.h.md) · [`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md) · [`smart_cover_planner_target_selector_inline.h`](smart_cover_planner_target_selector_inline.h.md) · [`stalker_movement_manager_smart_cover.cpp`](stalker_movement_manager_smart_cover.cpp.md)
**Tier floor** — T2: a planner node with a script callback; no layout or device concern

## Purpose

Declares the surface implemented in
[`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md).
The target selector is simultaneously an *action* inside the enclosing smart-cover
animation planner and a *planner* in its own right — the recurring nested-planner shape of
this engine, where one operator in a coarse plan expands into a finer plan when it runs.

Exported units:

- `target_selector` — the nested planner. Its goal state is "the planner has a target".
- `setup` — installs the evaluators and operators and seeds the initial world state.
- `update` — advances the nested plan, then fires the script callback.
- `object_name` — the label this node reports in planner traces.
- `callback` (set) / `callback` (get) — the script function invoked after each update, so
  that a smart-cover's Lua side can observe or override what the selector chose.

## State

```text
RECORD target_selector
  script_callback : optional<script function>   # called with the owning game object each update
  random          : random stream               # seeds the initial "looked out" belief
```
