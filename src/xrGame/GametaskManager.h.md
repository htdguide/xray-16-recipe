# src/xrGame/GametaskManager.h

> Declares the player's quest journal, implemented in [`GametaskManager.cpp`](GametaskManager.cpp.md).

**Needs** — [`GameTaskDefs.h`](GameTaskDefs.h.md) · [`Common/object_interfaces.h`](../Common/object_interfaces.h.md)
**Used by** — [`GameTask.cpp`](GameTask.cpp.md) · [`GametaskManager.cpp`](GametaskManager.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`UIGameSP.cpp`](UIGameSP.cpp.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`map_location.cpp`](map_location.cpp.md) · [`map_manager.cpp`](map_manager.cpp.md) · [`map_spot.cpp`](map_spot.cpp.md) · [`UIMapWnd.cpp`](ui/UIMapWnd.cpp.md) · [`UISecondTaskWnd.cpp`](ui/UISecondTaskWnd.cpp.md) · [`UITaskWnd.cpp`](ui/UITaskWnd.cpp.md) · [`map_hint.cpp`](ui/map_hint.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CGameTaskManager`, the single object permitted to change a task's state.
Substance is in [`GametaskManager.cpp`](GametaskManager.cpp.md).

Exported units:

- `GiveGameTaskToActor` — hand out a task, by authored identifier or as a task a script
  built. Two overloads; the identifier form constructs and delegates.
- `SetTaskState` — the only sanctioned way to move a task or objective to a new state.
  Two overloads, by identifier and by task.
- `UpdateTasks` — the once-a-frame re-evaluation sweep.
- `ActiveTask`, `SetActiveTask` — read and set the map's target for a task type.
- `HasGameTask` — look a task up by identifier or by the map marker it is showing.
- `GetGameTasks` — the journal, fetched from the alife registry on first use.
- `IterateGet`, `GetTaskIndex`, `GetTaskCount` — the PDA's paging and counting.
- `MapLocationRelcase` — the notification that a map marker is being destroyed
  elsewhere, so the journal and the map window drop their references.
- `AllowMultipleTask` — the per-game policy switch: one active task overall, or one per
  task type.
- `ActualFrame` — the frame the active-task derivation last ran, used by the UI as a
  cache key.
- `CleanupTasks`, `ResetStorage`, `DumpTasks` — clear the active identifiers, drop the
  cached journal handle after a load swaps the registry, and a debug dump.
- `m_gameTaskXml` — the authored task document, held open for the whole session and
  shared by every task that loads from it. Shared static state, not thread-safe; task
  construction is therefore single-threaded.
