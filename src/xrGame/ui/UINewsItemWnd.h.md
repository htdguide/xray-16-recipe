# src/xrGame/ui/UINewsItemWnd.h

> Declares one row of the PDA's news list: a timestamp, a caption, a body and an icon, sized to
> whichever is taller.

**Needs** — [`UINewsItemWnd.cpp`](UINewsItemWnd.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/XML/xrUIXmlParser.h`](../../xrUICore/XML/xrUIXmlParser.h.md)
**Used by** — [`UILogsWnd.cpp`](UILogsWnd.cpp.md) · [`UILogsWnd.h`](UILogsWnd.h.md) · [`UINewsItemWnd.cpp`](UINewsItemWnd.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UINewsItemWnd.cpp`](UINewsItemWnd.cpp.md).

## Exported units

- **The news row** — four pictures, of which only the icon is mandatory.
- `Init` — build from a named subtree of the PDA layout document.
- `Setup` — fill in from one news record and size the row.
- `Update` — deliberately empty: a news row never animates.
