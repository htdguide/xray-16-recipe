# src/xrGame/ui/UIItemInfo.h

> Declares the item description panel and the six sub-panels it composes.

**Needs** — [`UIItemInfo.cpp`](UIItemInfo.cpp.md) · [`UIWpnParams.h`](UIWpnParams.h.md) · [`ui_af_params.h`](ui_af_params.h.md) · [`UIOutfitInfo.h`](UIOutfitInfo.h.md) · [`UIBoosterInfo.h`](UIBoosterInfo.h.md) · [`UIInvUpgradeProperty.h`](UIInvUpgradeProperty.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) · [`UIActorMenu_action.cpp`](UIActorMenu_action.cpp.md) · [`UIInventoryUpgradeWnd.cpp`](UIInventoryUpgradeWnd.cpp.md) · [`UIItemInfo.cpp`](UIItemInfo.cpp.md) · [`UIMpTradeWnd_init.cpp`](UIMpTradeWnd_init.cpp.md) · [`UIMpTradeWnd_misc.cpp`](UIMpTradeWnd_misc.cpp.md) · [`UITalkDialogWnd.h`](UITalkDialogWnd.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIItemInfo.cpp`](UIItemInfo.cpp.md).

Nearly every member is **public**, because the screens that host this panel reposition and
re-text its fields themselves — the multiplayer buy menu overwrites the price, the trade
screen moves the note. That is not encapsulation lost; it is the panel being a *layout
container the host finishes*. A rebuild that seals it must give the hosts an explicit way to
override each field.

## Exported units

- **`CUIItemInfo`**
  - `InitItemInfo(xml)` — read the layout; **returns false for an empty document**.
  - `InitItemInfo(pos, size, xml)` — the same, with the caller supplying the rectangle.
  - `InitItem(cell item, compare item, price, trade note)` — the one filling entry point;
    every argument after the first is optional, and the price has a sentinel for "no price".
  - the six `TryAdd…` offers, public so a host can re-offer selectively.
  - `CurrentItem` — the subject.
  - `Draw` — skipped with no subject.
  - `delay` — the dwell, read from the layout and applied by the host.
  - `m_b_FitToHeight` — whether the panel shrinks to its content.
  - the field and sub-panel handles.
