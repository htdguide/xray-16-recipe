# src/xrGame/ui/UIInventoryUtilities.h

> Declares the chapter's shared services and, more importantly, the frozen icon-grid constants every screen addresses its atlases in.

**Needs** — [`UIInventoryUtilities.cpp`](UIInventoryUtilities.cpp.md) · [`inventory_item.h`](../inventory_item.h.md) · [`character_info_defs.h`](../../xrServerEntities/character_info_defs.h.md) · [`xrUICore/ui_defs.h`](../../xrUICore/ui_defs.h.md)
**Used by** — [`UIZoneMap.cpp`](../UIZoneMap.cpp.md) · [`encyclopedia_article.cpp`](../encyclopedia_article.cpp.md) · [`GameSpy_QR2_callbacks.cpp`](../gamespy/GameSpy_QR2_callbacks.cpp.md) · [`map_spot.cpp`](../map_spot.cpp.md) · [`UIActorInfo.cpp`](UIActorInfo.cpp.md) · [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuDeadBodySearch.cpp`](UIActorMenuDeadBodySearch.cpp.md) · [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) · [`UIActorMenuTrade.cpp`](UIActorMenuTrade.cpp.md) · [`UIArtefactPanel.cpp`](UIArtefactPanel.cpp.md) · [`UICellCustomItems.cpp`](UICellCustomItems.cpp.md) · [`UICharacterInfo.cpp`](UICharacterInfo.cpp.md) · [`UIDragDropReferenceList.cpp`](UIDragDropReferenceList.cpp.md) · [`UIHudStatesWnd.cpp`](UIHudStatesWnd.cpp.md) · _and 21 more_
**Tier floor** — T3.

## Purpose

Declares the surface implemented in
[`UIInventoryUtilities.cpp`](UIInventoryUtilities.cpp.md). The constants declared here are
the substantive part, because they are **frozen against the shipped game data** and nothing
else records them.

## Constants

```text
INV_GRID_WIDTH  = 50    # canvas units per equipment-icon grid cell
INV_GRID_HEIGHT = 50
ICON_GRID_WIDTH  = 64   # canvas units per character-portrait grid cell
ICON_GRID_HEIGHT = 64
CHAR_ICON      = 2 x 2 cells     # a portrait in the inventory and trade screens
CHAR_ICON_FULL = 2 x 5 cells     # a full-length portrait
TRADE_ICONS_SCALE = 4/5          # item icons on the trade screen only
```

An item's configuration section gives its icon position and size in **grid cells**; every
screen multiplies by these to get a canvas rectangle. Changing any of them re-cuts every
icon in all three games.

## Exported units

- **Packing and sorting** — `GreaterRoomInRuck` (a total order on items by descending
  footprint), `FreeRoom_inBelt` (would this item fit).
- **Atlases** — five accessors, each creating its material on first use, plus
  `CreateShaders` (a no-op) and `DestroyShaders`.
- **Clock formatting** — `GetGameDateAsString`, `GetGameTimeAsString`, `GetDateAsString`,
  `GetTimeAsString`, `GetTimeAndDateAsString`, `Get_GameTimeAndDate_AsString`,
  `GetTimePeriodAsString`, with the two precision enumerations.
- **Weight** — `UpdateWeight` (one widget, with inline colour markup), `UpdateWeightStr`
  (two widgets).
- **Threshold naming** — `GetRankAsText`, `GetReputationAsText`, `GetGoodwillAsText`,
  `ClearCharacterInfoStrings`.
- **Relation colours** — `GetGoodwillColor`, `GetReputationColor`, `GetRelationColor`.
- **Notifying the game** — `SendInfoToActor` (a UI act becomes an information portion),
  `SendInfoToLuaScripts` (the conversation screen's show/hide announcement).
