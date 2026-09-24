# src/xrGame/ui/UITradeBar.h

> Declares the trade screen's money-and-weight line.

**Needs** — [`UITradeBar.cpp`](UITradeBar.cpp.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuTrade.cpp`](UIActorMenuTrade.cpp.md) · [`UITradeBar.cpp`](UITradeBar.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UITradeBar.cpp`](UITradeBar.cpp.md).

Exported units:

- `CUITradeBar` — one side's trade summary line: a caption, a price and a weight limit.
- `init_from_xml(document, path)` — build from a named subtree.
- `UpdateData(price, weight)` — set both numbers and re-pack the line.
