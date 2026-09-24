# src/xrGame/ui/UIWarState.h

> Declares the PDA's faction-war status icon: a picture with a delayed tooltip, filled from script.

**Needs** — [`UIWarState.cpp`](UIWarState.cpp.md) · [`xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md)
**Used by** — [`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md) · [`UIFactionWarWnd.h`](UIFactionWarWnd.h.md) · [`UIWarState.cpp`](UIWarState.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIWarState.cpp`](UIWarState.cpp.md).

Exported units:

- `UIWarState` — one faction-war status slot: an icon and a dwell-delayed tooltip.
- `InitXML(document, path, parent)` — build and attach in one call.
- `UpdateInfo(icon, hint)` — show the slot with an icon and a tooltip; reports whether it took.
- `ClearInfo()` — hide the slot.
