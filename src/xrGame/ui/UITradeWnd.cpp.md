# src/xrGame/ui/UITradeWnd.cpp

> An emptied file: the trade screen's implementation was removed and only the precompiled-header
> include remains.

**Needs** — [`UITradeWnd.h`](UITradeWnd.h.md)
**Used by** — [`UITradeWnd.h`](UITradeWnd.h.md)
**Tier floor** — T4: the file contributes nothing to the build

## Purpose

Records that the trade screen used to be implemented here in native code and is not any more. The
file is still compiled, and produces nothing.

The trade screen the games actually show is assembled by script from the pieces this directory still
provides: two drag-and-drop lists with the placement and stack-splitting rules, the item info panel,
and [`CUITradeBar`](UITradeBar.cpp.md) for each side's money and weight. A rebuild should drop this
file rather than look for what was in it.

## State

`Stateless.`
