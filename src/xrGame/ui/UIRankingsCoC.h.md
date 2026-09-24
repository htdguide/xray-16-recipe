# src/xrGame/ui/UIRankingsCoC.h

> Declares the self-managing ranking row used by the PDA's ranking page.

**Needs** — [`UIRankingsCoC.cpp`](UIRankingsCoC.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIRankingWnd.cpp`](UIRankingWnd.cpp.md) · [`UIRankingWnd.h`](UIRankingWnd.h.md) · [`UIRankingsCoC.cpp`](UIRankingsCoC.cpp.md)
**Tier floor** — T3: a declaration of a screen element; nothing here constrains the language

## Purpose

Declares the surface implemented in [`UIRankingsCoC.cpp`](UIRankingsCoC.cpp.md): one row of the
ranking list, which decides for itself each frame whether it belongs in its parent list.

Exported units:

- `CUIRankingsCoC` — a ranking row bound to an index; constructed against the scroll container it
  will insert itself into.
- `init_from_xml(document, index, unique)` — build from one of two named layout templates.
- `Update` — the self-admission poll.
- `SetName` / `SetDescription` / `SetHint` / `SetIcon` — the four fields, fed from script.
- `DrawHint` — draw the tooltip only while the cursor is inside this row.
- `Reset` — withdraw from the list.
