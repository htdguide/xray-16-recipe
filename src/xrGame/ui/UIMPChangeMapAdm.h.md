# src/xrGame/ui/UIMPChangeMapAdm.h

> Declares the administrator's map-change page: a list of the maps legal for the running game
> type, a preview picture, a version label and a confirm button.

**Needs** — [`UIMPChangeMapAdm.cpp`](UIMPChangeMapAdm.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIMPAdminMenu.cpp`](UIMPAdminMenu.cpp.md) · [`UIMPAdminMenu.h`](UIMPAdminMenu.h.md) · [`UIMPChangeMapAdm.cpp`](UIMPChangeMapAdm.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMPChangeMapAdm.cpp`](UIMPChangeMapAdm.cpp.md). The
page is a plain container, not a dialog: it is one of the three pages the admin menu swaps
between, and it asks its parent to close the whole screen once it has acted.

## Exported units

- **The map-change page** — a window that is also a notification handler.
- `Init` — configure the five elements from the already-open layout document.
- `FillUpList` — repopulate the list from the map roster for the current game type.
- `OnItemSelect` — refresh the preview picture and the version label for the highlighted row.
- `OnBtnOk` — issue the level change and close the screen.
