# src/xrGame/ui/UIBuyWeaponTab.h

> Declares the buy menu's tab strip, which differs from the toolkit's in exactly one respect.

**Needs** — [`UIBuyWeaponTab.cpp`](UIBuyWeaponTab.cpp.md) · [`../../xrUICore/TabControl/UITabControl.h`](../../xrUICore/TabControl/UITabControl.h.md)
**Used by** — [`UIBuyWeaponTab.cpp`](UIBuyWeaponTab.cpp.md) · [`UIMpTradeWnd_init.cpp`](UIMpTradeWnd_init.cpp.md)
**Tier floor** — T3: one overridden notification

## Purpose

Declares the surface implemented in [`UIBuyWeaponTab.cpp`](UIBuyWeaponTab.cpp.md).

## `CUIBuyWeaponTab`

A tab strip for the multiplayer buy menu. It overrides only the notification handler; the
rest of the toolkit's tab control is inherited unchanged. The class body carries a large
commented-out earlier design — a stub tab, an active-state flag — which a rebuild should read
as: this class once did more and no longer does.
