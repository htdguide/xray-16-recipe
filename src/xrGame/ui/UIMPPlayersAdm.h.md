# src/xrGame/ui/UIMPPlayersAdm.h

> Declares the administrator's player page: a roster of connected clients and the per-player
> and all-player moderation actions.

**Needs** — [`UIMPPlayersAdm.cpp`](UIMPPlayersAdm.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIMPAdminMenu.cpp`](UIMPAdminMenu.cpp.md) · [`UIMPAdminMenu.h`](UIMPAdminMenu.h.md) · [`UIMPPlayersAdm.cpp`](UIMPPlayersAdm.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMPPlayersAdm.cpp`](UIMPPlayersAdm.cpp.md).

## Exported units

- **The player-administration page** — a window that is also a notification handler.
- `Init` — configure the roster, the buttons, the ping slider and the ban-duration drop-down.
- `RefreshPlayersList` — ask the multiplayer game layer for fresh player information.
- `FillPlayersList` — the continuation that consumes that answer and rebuilds the roster.
- `SetMaxPingLimit` / `SetMaxPingLimitText` — commit the ping ceiling, and render it.
- `GetSelPlayerScreenshot`, `GetSelPlayerConfig`, `KickSelPlayer`, `BanSelPlayer` — the four
  per-player actions.
