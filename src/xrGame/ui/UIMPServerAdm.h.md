# src/xrGame/ui/UIMPServerAdm.h

> Declares the administrator's server page: four alternative panels — a main menu, a weather
> picker, a game-type picker and a wall of match limits — of which exactly one is visible.

**Needs** — [`UIMPServerAdm.cpp`](UIMPServerAdm.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIMPAdminMenu.cpp`](UIMPAdminMenu.cpp.md) · [`UIMPAdminMenu.h`](UIMPAdminMenu.h.md) · [`UIMPServerAdm.cpp`](UIMPServerAdm.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMPServerAdm.cpp`](UIMPServerAdm.cpp.md).

## Exported units

- **The server-administration page** — a window that is also a notification handler.
- `Init` — configure the four panels and their sixty-odd controls from the layout document.
- `ShowChangeWeatherBtns`, `ShowChangeGameTypeBtns`, `ShowChangeGameLimitsBtns` — enter one
  of the three sub-panels.
- `OnBackBtn` — return to the main panel.
- `IsBackBtnShown` — reports whether a sub-panel is open, so the owning screen can decide
  what the cancel key means.
