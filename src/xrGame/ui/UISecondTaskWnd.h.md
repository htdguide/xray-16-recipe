# src/xrGame/ui/UISecondTaskWnd.h

> Declares the PDA's pop-over task list and the row that stands for one task in it.

**Needs** — [`UISecondTaskWnd.cpp`](UISecondTaskWnd.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UISecondTaskWnd.cpp`](UISecondTaskWnd.cpp.md) · [`UITaskWnd.cpp`](UITaskWnd.cpp.md)
**Tier floor** — T3: declarations of two screen elements

## Purpose

Declares the surface implemented in [`UISecondTaskWnd.cpp`](UISecondTaskWnd.cpp.md). Also fixes the
name of the layout document both the task list and the whole task page are read from —
`pda_tasks.xml` — which is shipped data and therefore frozen.

Exported units:

- `UITaskListWnd` — the pop-over list of in-progress tasks: opening it takes the keyboard and the
  navigation focus, closing it gives both back.
  - `init_from_xml(document, path)` — build from a named subtree.
  - `ShowOnlySecondaryTasks(mode)` — whether storyline tasks are filtered out of the list.
  - `UpdateList()` — rebuild the rows from the task manager.
- `UITaskListWndItem` — one row: task title, a map-spot toggle, a focus-the-map button, and a
  storyline/secondary marker.
  - `init_task(task, parent)` — bind the row to a task and build it.
  - `get_priority_task()` — the sort key the list orders rows by.
  - `Focus()` — put the navigation focus on this row's title.
