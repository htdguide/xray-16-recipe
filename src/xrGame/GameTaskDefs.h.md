# src/xrGame/GameTaskDefs.h

> The vocabulary of the quest system: task state, task type, the identifiers a task and its objectives are addressed by, and the saved registry that carries the player's task list.

**Needs** — [`alife_abstract_registry.h`](alife_abstract_registry.h.md) · [`GameTask.h`](GameTask.h.md)
**Used by** — [`GameTask.cpp`](GameTask.cpp.md) · [`GameTask.h`](GameTask.h.md) · [`GametaskManager.cpp`](GametaskManager.cpp.md) · [`GametaskManager.h`](GametaskManager.h.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`UISecondTaskWnd.cpp`](ui/UISecondTaskWnd.cpp.md) · [`UITaskWnd.h`](ui/UITaskWnd.h.md)
**Tier floor** — T2: enumerations and a serialized container; the frozen part is the save layout

## Purpose

A constants-and-types header with no implementation file, so it is the substance holder
for the quest system's shared vocabulary. Three decisions live here.

**A task is addressed by a text identifier, an objective by a small index.** Tasks come
from an authored XML file and are referenced by name from scripts and from dialogue, so
their identifier must be stable across a save. Objectives are positional within their
task, so a narrow integer suffices — and index zero is reserved for the task *itself*,
which is modelled as its own root objective rather than as a separate kind of thing.

**States and types are small closed enumerations with an explicit "not set" member.** The
dummy value is not a placeholder to be tidied away: a task newly constructed from a save
or from script has no state yet, and the code distinguishes "never started" from "in
progress". A rebuild that models the state as an optional gets the same effect.

**The player's task list is part of the alife registry**, not a free-standing structure.
It is keyed by owner identifier, which is why a task list belongs to an inventory owner
and not to the world: in principle every character could carry one, in practice only the
player does.

## State

```text
ENUM TaskState
  fail                # the objective can no longer be achieved
  in_progress
  completed
  not_set             # constructed but never started; distinct from all three above

ENUM TaskType
  storyline           # the main plot; the default an unclassified task falls back to
  additional          # a side quest
  insignificant       # background flavour
  not_set

RECORD TaskIdentifiers
  task_id      : text        # the authored name; stable across saves and scripts
  objective_id : int (16-bit)  # 0 means the task itself
  # invariant: objective 0 always exists; objectives 1..n are the sub-goals

RECORD TaskListEntry
  task_id   : text     # duplicated from the task so the list can be indexed without loading
  task      : GameTask
  # invariant on save: entry.task_id == entry.task.id — asserted, not repaired

RECORD TaskRegistry                       # one per inventory owner, keyed by entity identifier
  entries : map<int, list<TaskListEntry>>
  active  : list<text>                    # one active task identifier per task type
```

## `ROOT_TASK_OBJECTIVE`

**Contract** — the reserved objective index zero, meaning the task itself. Every place
that walks a task's objectives must decide whether to include it; the count accessor
takes a flag for exactly that reason.

## `g_active_task_id`

**Contract** — one *active* task identifier per task type: the task the map and the PDA
highlight for that category. A global rather than a field of the task list because the UI
reads it without holding an owner, and because its lifetime is the session rather than
the owner.

**Invariants** — saved and restored alongside the task registry, and in the same stream:
a restored session must highlight the same task it highlighted when saved.

## `SGameTaskKey`

**Contract** — one entry in an owner's task list: the task's identifier and the task
itself. Serializes as the task alone; the identifier is recovered from the loaded task,
which is why the save path asserts that the two agree rather than writing both.

**Notes** — the entry owns its task and destroys it on demand through a separate
destruction hook rather than at scope exit. That is a consequence of the registry's
bulk-clear model — the whole list is discarded at once when a session ends — and a
rebuild with ordinary ownership does not need the hook.

## `CGameTaskRegistry`

**Contract** — the saved per-owner task list, built on the generic alife registry. Its
only addition to the generic behaviour is that it writes and reads the active-task
identifiers immediately after the registry body.

**Invariants** — the active identifiers are appended *after* the registry's own payload
and read in the same order, so the two are one atomic save record. Splitting them would
let a restored session hold a highlighted task identifier that no longer names a task in
the list.

## Notes

The comment history in the source records that at one point every task was forced to the
storyline type and that the change was later reverted. The three types therefore all
carry meaning again, and the "insignificant" type in particular exists only so the PDA
can hide a class of tasks from the main list.
