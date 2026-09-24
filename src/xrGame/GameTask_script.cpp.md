# src/xrGame/GameTask_script.cpp

> Exports the quest classes to the script layer, and supplies the two tree-building calls that only exist for scripts.

**Needs** — [`GameTask.h`](GameTask.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a registration table plus two one-line methods

## Purpose

The script-export sibling of [`GameTask.cpp`](GameTask.cpp.md). It is separate only
because the binding layer's registration machinery is expensive to compile; nothing here
is a design decision except the *shape of the exported surface*, which is frozen by
conformance criterion 10 — the shipped scripts build quests through exactly these names.

## `AddObjective_script`

**Contract** — append a script-built objective to a task, resolving its script-added
condition names into callable handles first. The objective is **copied** into the task's
list and the script's handle to it is adopted — the task now owns it.

**Invariants** — resolution happens at append time, not at construction, because a script
adds conditions after constructing the objective. An objective appended before its
conditions are set gets no conditions.

## `GetObjective_script`

**Contract** — hand the script a reference to one objective by index, with index zero
meaning the task itself. Non-owning: the task keeps the objective.

## `script_register`

**Contract** — registers three things with the script virtual machine:

- a namespace `task` carrying two enumerations, `task_state` (`fail`, `in_progress`,
  `completed`, `task_dummy`) and `task_type` (`storyline`, `additional`,
  `insignificant`). The names are frozen; the values are the engine's own;
- `SGameTaskObjective`, constructible from (parent task, index), with read/write
  properties for title, description, type, icon name, article identifier and key, map
  hint, map location kind and map object identifier; the read-only index and state; the
  marker calls `create_map_location`, `change_map_location`, `remove_map_locations`; and
  the eight condition-adding calls — complete/fail crossed with checked
  (`add_complete_info`, `add_complete_func`, `add_fail_info`, `add_fail_func`) and fired
  (`add_on_complete_*`, `add_on_fail_*`);
- `CGameTask`, derived from the objective so it inherits every property above,
  additionally exposing `load` (build from the authored definition), the identifier and
  priority, `add_objective`, `get_objective` and `get_objectives_cnt`.

**Notes** — three compatibility details are load-bearing and would be invisible in a
clean rewrite:

- `set_object_id` is registered as a second name for `set_map_object_id`, because one
  shipped game's scripts use the older name.
- `get_objectives_cnt` is registered twice, once taking the "exclude the root" flag and
  once taking nothing and meaning "include the root". Overload resolution by argument
  count is what makes both call forms work.
- `add_objective` transfers ownership of the objective to the task. A rebuild whose
  binding layer cannot express that must arrange the same lifetime some other way; a
  script that keeps using its handle after appending is reading the task's copy.
