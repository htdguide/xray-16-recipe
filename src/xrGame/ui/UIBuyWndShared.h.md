# src/xrGame/ui/UIBuyWndShared.h

> Selects which buy menu the build uses, and declares the price-and-slot table the menu is
> built from.

**Needs** — [`UIBuyWndShared.cpp`](UIBuyWndShared.cpp.md) · [`Restrictions.h`](Restrictions.h.md)
**Used by** — [`game_cl_deathmatch.h`](../game_cl_deathmatch.h.md) · [`game_sv_artefacthunt.cpp`](../game_sv_artefacthunt.cpp.md) · [`game_sv_deathmatch.cpp`](../game_sv_deathmatch.cpp.md) · [`game_sv_mp.cpp`](../game_sv_mp.cpp.md) · [`game_sv_teamdeathmatch.cpp`](../game_sv_teamdeathmatch.cpp.md) · [`UIBuyWndShared.cpp`](UIBuyWndShared.cpp.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md)
**Tier floor** — T3: a sorted table over configuration

## Purpose

Two unrelated things share this header because both are needed by everything that touches the
buy menu.

## The implementation switch

A build-time selection names one class as *the* buy menu. Only the newer implementation is
reachable; the older one is not built. A rebuild has one buy menu and needs no switch — but
should know the alias exists, because call sites name the alias rather than the class.

## `CItemMgr` — the item catalogue

Declares the surface implemented in [`UIBuyWndShared.cpp`](UIBuyWndShared.cpp.md). The table
that answers, for every buyable item: what it costs at each of the five ranks, and which of
the menu's groups it belongs in.

- `Load(price section)` — build the table.
- `GetItemCost(section, rank)` / `GetItemSlotIdx(section)` — the two lookups.
- `GetItemIdx(section)` / `GetItemName(index)` / `GetItemsCount()` — the table as an ordered
  sequence, which is what gives every item a stable index.
- `Dump()` — print the table and assert it is complete.

**The ordering is load-bearing.** The table is kept sorted by section name, and an item's
*position* in that order is used as its identity elsewhere — so two machines with the same
configuration derive the same indices. A rebuild that uses an unordered map breaks that
agreement silently.
