# src/xrGame/ui/UIMapLegend.h

> Declares the map legend pop-up and the one row inside it: a picture (or up to four) beside a
> line of explanatory text.

**Needs** — [`UIMapLegend.cpp`](UIMapLegend.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIMapLegend.cpp`](UIMapLegend.cpp.md) · [`UITaskWnd.cpp`](UITaskWnd.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMapLegend.cpp`](UIMapLegend.cpp.md).

## Exported units

- **The legend pop-up** — a framed panel with a caption, a close button and a scrolling list.
- **The legend row** — one to four marker images plus a caption, sized to its text.
- `init_from_xml` on each — build from a named subtree of the map screen's layout document.
- `Show`, `SendMessage` — visibility and the close button.
