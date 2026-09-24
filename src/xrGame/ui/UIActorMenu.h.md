# src/xrGame/ui/UIActorMenu.h

> Declares the one screen behind the inventory, the trade counter, the upgrade bench and the
> loot panel — four modes of a single window — and names the two enumerations that every one
> of its rules is written against.

**Needs** — [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`../../xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md) · [`../../xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md) · [`../../xrServerEntities/inventory_space.h`](../../xrServerEntities/inventory_space.h.md)
**Used by** — [`ActorInput.cpp`](../ActorInput.cpp.md) · [`UIGameCustom.cpp`](../UIGameCustom.cpp.md) · [`UIGameDM.cpp`](../UIGameDM.cpp.md) · [`UIGameSP.cpp`](../UIGameSP.cpp.md) · [`eatable_item.cpp`](../eatable_item.cpp.md) · [`game_cl_artefacthunt.cpp`](../game_cl_artefacthunt.cpp.md) · [`game_cl_capture_the_artefact.cpp`](../game_cl_capture_the_artefact.cpp.md) · [`game_cl_deathmatch.cpp`](../game_cl_deathmatch.cpp.md) · [`game_cl_teamdeathmatch.cpp`](../game_cl_teamdeathmatch.cpp.md) · [`script_game_object.cpp`](../script_game_object.cpp.md) · [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuDeadBodySearch.cpp`](UIActorMenuDeadBodySearch.cpp.md) · [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) · _and 8 more_
**Tier floor** — T2: a screen owning a dozen widget trees and a mode machine

## Purpose

Declares the surface implemented across eight files. The substance is split by concern, not by
size, and the split is worth knowing before reading any of them:

| File | Carries |
|---|---|
| [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) | construction from layout documents; the two layout dialects; the drop-permission table; callback wiring |
| [`UIActorMenu.cpp`](UIActorMenu.cpp.md) | the mode machine; per-frame update; list↔type mapping; the current item; the highlight rules |
| [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) | filling the lists from an inventory; the move operations; the context menu; **how a UI action becomes a game event** |
| [`UIActorMenu_action.cpp`](UIActorMenu_action.cpp.md) | the gesture handlers — drop, double click, right click, focus — and keyboard bindings |
| [`UIActorMenuTrade.cpp`](UIActorMenuTrade.cpp.md) | trade mode: the four lists, pricing, the trade transaction |
| [`UIActorMenuDeadBodySearch.cpp`](UIActorMenuDeadBodySearch.cpp.md) | loot mode: corpses and containers, take-all, information transfer |
| [`UIActorMenuUpgrade.cpp`](UIActorMenuUpgrade.cpp.md) | upgrade mode: selecting an item to work on |
| [`UIActorMenu_script.cpp`](UIActorMenu_script.cpp.md) | repair and upgrade decisions delegated to script; the frozen script surface |

## `EDDListType` — the nine roles a list can play

The **drop-permission table and every move rule are written against this enumeration, not
against list identity**. A list's identity says which widget it is; its type says what it
means. `iActorSlot`, `iActorBag`, `iActorBelt`, `iActorTrade`, `iPartnerTradeBag`,
`iPartnerTrade`, `iDeadBodyBag`, `iQuickSlot`, `iTrashSlot`, plus `iInvalid`.

The mapping is many-to-one and mode-dependent — three different widgets are `iActorBag` in
three different modes — which is what lets one set of rules serve four screens. The
enumeration is **exported to script** and therefore frozen.

## `EMenuMode` — the four modes plus undefined

`mmInventory`, `mmTrade`, `mmUpgrade`, `mmDeadBodySearch`, and `mmUndefined` for "closed".
Every mode transition runs an exit step for the old mode and an entry step for the new; see
[`UIActorMenu.cpp`](UIActorMenu.cpp.md).

## `CUIActorMenu`

The screen. Notable structure, because the count is itself information:

- **Sixteen drag-and-drop lists** in one array indexed by an internal list enumeration, plus
  a separate quick-slot list. Several entries alias the same widget.
- **Two participants**: the actor's inventory owner, and either a partner inventory owner or
  an inventory container — never both.
- **Two trade handles**, one per side, live only while trade mode is entered.
- **Per-mode duplicates** of the character-info panel and the item-info panel, because the two
  layout dialects place them differently; a mode-specific accessor picks the right one.
- One context menu, two message boxes, one hint window, one trash target.

Nearly every widget pointer may be absent: see the layout dialect discussion in
[`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md).

## The sound set

Ten named sounds — open, close, item to slot, to belt, to bag, context menu, drop, attach,
detach, use — all loaded from the layout document rather than hard-coded, and all optional.
