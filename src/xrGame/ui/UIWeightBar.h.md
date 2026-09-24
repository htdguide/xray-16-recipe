# src/xrGame/ui/UIWeightBar.h

> Declares the carried-weight line used by the inventory and trade screens.

**Needs** — [`UIWeightBar.cpp`](UIWeightBar.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIActorMenuDeadBodySearch.cpp`](UIActorMenuDeadBodySearch.cpp.md) · [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuTrade.cpp`](UIActorMenuTrade.cpp.md) · [`UIWeightBar.cpp`](UIWeightBar.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIWeightBar.cpp`](UIWeightBar.cpp.md).

Exported units:

- `CUIWeightBar` — the line: a caption, the current weight, and optionally the limit.
- `init_from_xml(document, prefix)` — build from three elements named by a caller-supplied prefix.
- `UpdateData(weight)` — a bare weight.
- `UpdateData(owner)` — an inventory owner's weight and limit, plus two optional carry-capacity
  indicators.
- `m_BagWnd`, `m_BagWnd2` — those indicators, set by the owning screen rather than built here.
