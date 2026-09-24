# src/xrGame/ui/UIActorInfo.h

> Declares the personal-terminal page that shows the player's own statistics: a master list of
> categories and a detail list for the selected one.

**Needs** — [`UIActorInfo.cpp`](UIActorInfo.cpp.md) · [`../../xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIActorInfo.cpp`](UIActorInfo.cpp.md) · [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md)
**Tier floor** — T3: two scrolling lists filled from a statistics registry

## Purpose

Declares the surface implemented in [`UIActorInfo.cpp`](UIActorInfo.cpp.md).

## `CUIActorInfoWnd`

The page. A master-detail pair: a scrolling list of statistic categories on the left beside
the player's own character panel, and a scrolling list of that category's entries on the right.

- `Init()` — build from a layout document; returns false when the document is absent, which is
  how the page is omitted in games that do not have it.
- `Show(status)` — on being shown, re-reads the player's character record and refills the
  master list. It refreshes on every show rather than on change.
- `FillPointsDetail(id)` — fill the detail list for one category.
- `DetailList()` / `MasterList()` — the two lists, exposed because the row widgets drive them.

## `CUIActorStaticticHeader`

One master-list row: two text fields and a category identifier. Selectable, and **selecting
it is what fills the detail list** — the row calls back into the page. Selection is shown by
raising the first field's text alpha to full and restoring it to the authored value on
deselection, so the authored colour is remembered per row rather than assumed.

## `CUIActorStaticticDetail`

One detail-list row: four text fields — an index, a name, a count or value, and a point total.
