# src/xrGame/ui/UITaskWnd.h

> Declares the PDA's task page — the map plus one or two active-task panels — and the panel itself.

**Needs** — [`UITaskWnd.cpp`](UITaskWnd.cpp.md) · [`../GameTaskDefs.h`](../GameTaskDefs.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIMapWnd2.cpp`](UIMapWnd2.cpp.md) · [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md) · [`UITaskWnd.cpp`](UITaskWnd.cpp.md)
**Tier floor** — T3: declarations of a page and its task panel

## Purpose

Declares the surface implemented in [`UITaskWnd.cpp`](UITaskWnd.cpp.md).

Exported units:

- `CUITaskWnd` — the task page: owns the map, one or two active-task panels, the pop-over task list,
  the map legend and the map filters, and is the **notification hub** that every task-related
  message in the PDA passes through.
  - `Init()` — build; returns failure when the layout document is absent, so the PDA can omit the
    page entirely.
  - `ReloadTaskInfo()` — re-read the active tasks and re-apply every map-spot filter.
  - `Show_TaskListWnd` / `Switch_ShowMapLegend` — the two pop-overs.
  - Four filter queries and four filter setters, forwarded to the filter widget, each answering
    "true" when there is no filter widget at all.
  - `DrawHint()` — draw the map's tooltip above everything.
- `CUITaskItem` — one active-task panel: icon, caption, and a dwell-timed tooltip.
