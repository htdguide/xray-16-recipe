# src/xrGame/GametaskManager.cpp

> The player's quest journal: hands tasks out, re-evaluates every in-progress objective once a frame, decides which task is the *active* one per category, and keeps the map pointer on it.

**Needs** — [`GametaskManager.h`](GametaskManager.h.md) · [`GameTask.h`](GameTask.h.md) · [`GameTaskDefs.h`](GameTaskDefs.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`ui/UIPdaWnd.h`](ui/UIPdaWnd.h.md) · [`ui/UIMapWnd.h`](ui/UIMapWnd.h.md) · [`encyclopedia_article.h`](encyclopedia_article.h.md)
**Used by** — reached through its declarations in [`GametaskManager.h`](GametaskManager.h.md); callers name that, not this file.
**Tier floor** — T2: list bookkeeping and a once-a-frame sweep over a small collection

## Purpose

Exactly one of these exists while a session runs, owned by the player. It is the only
thing that may change a task's state, and that monopoly is the point: task state changes
have side effects (map markers, information portions, script calls, the PDA's unread
badge) and centralising them means those side effects fire once and in one order.

The manager's other job is choosing the **active** task — the one the map points at.
There is one active task per task type, and which task wins is decided by *priority*,
lowest number first. A rebuild must keep the rule that the active task is derived, not
stored as a flag on the task: the derivation is re-run whenever the list changes, so
completing the pointed-at quest silently promotes the next one.

## State

```text
RECORD TaskManager
  tasks        : list<TaskListEntry>   # the player's journal, from the saved alife registry
  changed      : bool                  # the active-task derivation is stale
  multiple     : bool                  # whether more than one task type may be active at once
  actual_frame : int                   # the frame the derivation last ran; the UI's cache key
  quest_document : XmlDocument         # the authored task definitions, held open for the session
```

Invariants:

- The journal is kept **sorted by priority descending** at all times — after every
  addition and at every re-derivation. Every "first matching task" query relies on this
  and does no sorting of its own.
- A task identifier appears at most once in the journal *in progress*. The same
  identifier may appear again once the earlier instance has finished, which is how
  repeatable quests work.
- The active identifier for a type either names an in-progress task in the journal or is
  empty. It is never left pointing at a finished task; the state setter clears it.
- The journal is not owned here. It lives in the alife registry keyed by the player's
  identifier, so it is saved and restored with the world rather than with this object.

## `CGameTaskManager` construction and destruction

**Contract** — opens the authored task document for the whole session and binds to the
player's journal in the alife registry. On construction it re-establishes the active task
for every type from the identifiers the save restored, dropping any that no longer names
an in-progress task. Destruction releases the journal wrapper, clears the active
identifiers and closes the document.

**Invariants** — the document is loaded once and shared, and the source notes it is not
safe to touch from more than one thread. Task construction reads it, so task construction
is single-threaded.

**Notes** — the journal wrapper is initialized with entity identifier zero rather than
the player's actual identifier. In single player the player is always entity zero; this
is a hard-coded assumption that a rebuild should make explicit.

## `GiveGameTaskToActor`

**Contract** — hand a task to the player, either by authored identifier (constructing it)
or as an already-built task from script. Refuses and reports when the identifier is
already in the journal *in progress*. Stamps the receive time, the deadline and the map
timer from the current **game** clock, inserts the task in priority order, marks it
arrived, updates which task is active, and refreshes the PDA.

**Invariants** — the deadline and the marker countdown are both expressed as absolute
game times derived from durations in *seconds* supplied by the caller; the internal
representation is milliseconds. A duration of zero therefore encodes "deadline equals
receive time", which the objective reads as "no deadline". That coincidence is the
encoding, not an accident, and a rebuild must preserve it or introduce an explicit
optional.

```text
FUNCTION give_task(task, complete_seconds, timer_seconds)
  commit any script-added conditions on the task     # resolve names to callable handles
  IF a task with this identifier is already in progress THEN
    report the conflict and RETURN none
  now = game time
  task.receive_time     = now
  task.time_to_complete = now + complete_seconds
  task.timer_finish     = now + timer_seconds
  append to journal, then stable sort by priority descending
  task.on_arrived()                                  # state, encyclopedia, map marker
  IF only one task may be active THEN
    make it active
  ELSE IF its type is storyline or additional THEN
    make it active IF no task of that type is active, or the active one has a worse priority
  refresh the PDA's unread badge
  fire the task's state-change callback
```

**Notes** — the duplicate check on the identifier-taking form uses "exists at all",
while the check inside the common path uses "exists in progress". The looser inner check
is what lets a repeatable quest be re-given; the stricter outer one is a caller
convenience that can be switched off. The two together are confusing and a rebuild should
pick one policy.

An insignificant task never becomes active even when nothing else is. Only storyline and
additional tasks compete for the map pointer.

## `SetTaskState`

**Contract** — the single entry point for moving a task or one of its objectives to a new
state. Applies the state through the task (which cascades to in-progress sub-objectives
when the root is addressed), then re-derives the active task and advances the active
objective, then refreshes the PDA.

**Invariants** — this is where the *journal* reacts to a task ending, and the two rules
are:

- If the root was addressed, or the task has no objective left in progress, and this task
  was the active one, the active identifier for its type is cleared. The next
  re-derivation will pick a successor.
- Otherwise, if the objective that just ended was the one the map pointed at and it was
  not the last one, the pointer advances to the **next objective by index**. Objectives
  are therefore a linear sequence, not a set: a quest's steps are walked in authored
  order.

```text
FUNCTION set_task_state(task, state, objective)
  changed = true
  type = IF multiple THEN task.type ELSE storyline
  task.set_state(state, objective)
  IF (objective is the root OR task has no objective in progress) AND task is active THEN
    clear the active identifier for type
  ELSE IF objective is not the root
       AND objective is the active one
       AND objective is not the last one THEN
    task.active_objective = objective + 1
  refresh the PDA
```

**Notes** — the identifier-taking overload looks the task up with *different strictness
depending on whether an objective was named*: naming an objective searches only
in-progress tasks, addressing the root searches all of them. That asymmetry exists so a
script may re-fail an already-finished task at the root but may not reach into a finished
task's steps.

## `UpdateTasks`

**Contract** — the once-a-frame sweep. Skipped entirely while paused. Disables every map
pointer, re-evaluates each in-progress objective of each in-progress task, applies any
state the evaluation implies, then re-enables the pointer on the marker of each active
task, and re-derives active tasks if anything changed.

**Invariants** — the sweep iterates over a **snapshot** of the journal, because applying
a state change can append to, reorder, or remove entries from the real list (a script
called on completion may give a new task). Iterating the live list would be undefined.
The snapshot is shallow: the tasks themselves are shared with the live list, so state
applied through the snapshot is visible.

The pointer is disabled for *all* markers first and then re-enabled only for the active
tasks'. That is a clear-and-rebuild rather than a diff, and it is why the quest system
never leaves an orphaned pointer on the map.

```text
FUNCTION update_tasks()
  IF paused THEN RETURN
  map.disable_all_pointers()
  IF journal is empty THEN RETURN
  snapshot = copy of the journal                  # applying a state may mutate the journal
  FOR EACH entry IN snapshot
    IF entry.task.state is not in_progress THEN CONTINUE
    FOR EACH objective index i IN entry.task, root included
      obj = entry.task.objective(i)
      IF obj.state is not in_progress THEN CONTINUE
      implied = obj.update_state()                # pure; see GameTask.cpp
      IF implied is fail or completed THEN set_task_state(entry.task, implied, i)
  FOR EACH task type
    IF a task is active for that type THEN enable the pointer on its linked marker
  IF changed THEN re_derive_active_tasks()
```

**Notes** — the snapshot is taken on the stack with a size known at entry. That is a
frame-budget decision (this runs every frame and the journal is small), not a semantic
one; a rebuild may allocate normally.

Note that the root objective is evaluated *first* in the inner loop, and a root failure
cascades to the sub-objectives immediately. The remaining iterations then skip them
because they are no longer in progress. Reversing the loop order would let a
sub-objective complete on the same frame its parent failed.

## `UpdateActiveTask`

**Contract** — re-sort the journal by priority and, for every type that has no active
task, promote the highest-priority in-progress task of that type. Records the frame it
ran on, which the PDA uses as a cache key to know when to rebuild its list. Clears the
changed flag.

**Invariants** — promotion happens for the storyline type always, and for the other types
only when multiple active tasks are allowed. This is the one place the "no active quest"
state is repaired, so any code that clears an active identifier can rely on a successor
being chosen at most one frame later.

## `ActiveTask`

**Contract** — the active task of a type, or none. Returns none for any type but
storyline when multiple active tasks are not allowed, and none when the stored identifier
does not name an in-progress task — so a stale identifier reads as "no active task"
rather than as a dangling one.

## `SetActiveTask`

**Contract** — make a task the map's target, optionally pointing at a specific objective.
Stores its identifier under its type (or under storyline when only one may be active),
sets the task's active objective, marks the task read, and flags the derivation stale.

**Notes** — making a task active marks it *read*, which clears the PDA's unread badge.
That couples "the player looked at the map" to "the player has seen this quest", which
is a deliberate simplification.

## `IterateGet`

**Contract** — step through the journal from a given task to the next (or previous) one
matching a state and type; with no starting task, return the first match. Used by the PDA
to page through quests in one category.

**Notes** — written recursively, descending once per non-matching neighbour, and the
recursion is unbounded in the journal's length. A rebuild should write the obvious loop;
nothing about the recursion is load-bearing.

## `GetTaskIndex` / `GetTaskCount`

**Contract** — the one-based position of a task among those matching a state and type,
and how many match. Both scan the journal. They exist for the PDA's "3 of 7" display and
rely on the journal's priority ordering for the numbering to be stable.

**Notes** — the index returns zero both for "not found" and for a null task, so the
caller cannot distinguish the two. A rebuild should return an optional.

## `HasGameTask`

**Contract** — two lookups: by identifier, and by map marker. The marker form asks each
task for its *linked* marker, which for a task with sub-objectives is the active
objective's marker — so a task is found by the marker it is currently showing, not by
every marker it ever created.

## `MapLocationRelcase`

**Contract** — called when a map marker is about to be destroyed by someone else. Tells
the map window to forget it, then finds the task that owns it and clears the task's
marker reference without asking the map manager to remove it again.

**Invariants** — this is the dangling-reference sweep the preface's runtime invariants
demand: a destroyed marker must be unreferenced by the journal and by the UI before its
memory is released. A rebuild with a different ownership model still needs the
notification, because a marker may be destroyed by the map manager for reasons the quest
system knows nothing about (its target entity despawned).

## `CleanupTasks`, `ResetStorage`, `AllowMultipleTask`, `ActualFrame`, `DumpTasks`

**Contract** — clear the active identifiers; drop the cached handle on the journal so it
is re-fetched from the registry (used when the registry is swapped under the manager by a
load); the policy switch for one-active-task-per-type; the frame stamp the UI caches
against; and a console dump of the journal for debugging.

## Notes

The "multiple tasks" policy is a per-game difference: one shipped game allows a single
active quest and the others allow one per category. It is set from outside rather than
read from configuration here.

Two dead code paths remain in the source: a commented-out overload that set the active
task by identifier, and an unconditional duplicate check whose condition has been
commented out. Neither affects behaviour; both are noted only so a reader comparing the
recipe against the source is not looking for a decision that is not there.
