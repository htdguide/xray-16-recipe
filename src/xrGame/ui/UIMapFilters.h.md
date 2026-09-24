# src/xrGame/ui/UIMapFilters.h

> Declares the map's marker filter panel: four check boxes that decide which kinds of map
> location are drawn, plus the keyboard mode that makes them reachable without a pointer.

**Needs** — [`UIMapFilters.cpp`](UIMapFilters.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIMapFilters.cpp`](UIMapFilters.cpp.md) · [`UITaskWnd.cpp`](UITaskWnd.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMapFilters.cpp`](UIMapFilters.cpp.md).

## Exported units

- **The filter panel** — a window holding one check box per filter kind.
- `eSpotsFilter` — the four filterable kinds: treasures, quest-giving characters, secondary
  tasks and primary objects, plus a count. The order is the tab order.
- `Init` — build whichever check boxes the layout document defines; reports whether any exist.
- `Activate` — enter or leave the panel's keyboard mode.
- `IsFilterEnabled` / `SetFilterEnabled` — query and set one filter. A filter whose check box
  the document omitted reads as **enabled**.
- `Reset`, `OnKeyboardAction`, `SendMessage` — the lifecycle and event hooks.
