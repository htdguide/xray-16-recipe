# src/xrGame/ui/Restrictions.h

> Declares the multiplayer buy menu's two gates: which items a rank may buy at all, and how
> many of each group it may carry.

**Needs** — [`Restrictions.cpp`](Restrictions.cpp.md)
**Used by** — [`Restrictions.cpp`](Restrictions.cpp.md) · [`UIBuyWndShared.cpp`](UIBuyWndShared.cpp.md) · [`UIBuyWndShared.h`](UIBuyWndShared.h.md) · [`UIMpItemsStoreWnd.cpp`](UIMpItemsStoreWnd.cpp.md) · [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) · [`UIMpTradeWnd_items.cpp`](UIMpTradeWnd_items.cpp.md) · [`UIMpTradeWnd_misc.cpp`](UIMpTradeWnd_misc.cpp.md) · [`UIMpTradeWnd_trade.cpp`](UIMpTradeWnd_trade.cpp.md) · [`UIMpTradeWnd_wpn.cpp`](UIMpTradeWnd_wpn.cpp.md)
**Tier floor** — T3: table lookups over configuration

## Purpose

Declares the surface implemented in [`Restrictions.cpp`](Restrictions.cpp.md).

## `get_rank`

**Contract** — Free function. Given an item's configuration section name, return the lowest
rank that is allowed to buy it.

## `CRestrictions`

The gate itself, one process-wide instance. Holds the item-group table and, per rank, a count
limit per group.

- `InitGroups()` — build both tables from configuration; idempotent.
- `SetRank(r)` / `GetRank()` / `GetRankName(r)` — the rank all queries are answered against.
- `IsAvailable(section)` — may the current rank buy this item at all.
- `GetItemGroup(section)` — which group an item belongs to.
- `GetItemCount(section)` / `GetGroupCount(group)` — how many of that item's group the current
  rank may carry.

## `RESTR`

A parsed `name:count` pair from the configuration's restriction lists.

## The rank count

The number of ranks is **five**, fixed at build time, and the value appears in the
configuration as section names `rank_0` through `rank_4` plus a sixth, `rank_base`. It is
frozen by the shipped data.
