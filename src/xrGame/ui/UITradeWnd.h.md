# src/xrGame/ui/UITradeWnd.h

> An emptied file: the trade screen's declaration was removed and the header is now a single
> include guard.

**Needs** — [`UITradeWnd.cpp`](UITradeWnd.cpp.md)
**Used by** — [`UITradeWnd.cpp`](UITradeWnd.cpp.md)
**Tier floor** — T4: the file contributes nothing to the build

## Purpose

Records that the trade screen's old declaration used to live here. The file still appears in the
module's source list and still declares its include guard, and declares nothing else — the trade
screen is now a script built on the chapter-15 widget toolkit and the screen pieces this directory
still ships ([`CUITradeBar`](UITradeBar.cpp.md), the drag-and-drop lists, the item info panel).

One reference to the removed type survives: the single-player game UI still forward-declares it. A
rebuild should remove both.

## State

`Stateless.` Nothing is declared.
