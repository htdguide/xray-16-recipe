# src/xrGame/ui/UIDragDropReferenceList.h

> Declares the quick-use bar: a cell board over a persistent table of item section names.

**Needs** — [`UIDragDropReferenceList.cpp`](UIDragDropReferenceList.cpp.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`xrServerEntities/inventory_space.h`](../../xrServerEntities/inventory_space.h.md)
**Used by** — [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) · [`UIDragDropReferenceList.cpp`](UIDragDropReferenceList.cpp.md) · [`UIHelper.cpp`](UIHelper.cpp.md) · [`UIXmlInit.cpp`](UIXmlInit.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in
[`UIDragDropReferenceList.cpp`](UIDragDropReferenceList.cpp.md).

## Exported units

- **`CUIDragDropReferenceList`** — the quick-use bar.
  - `Initialize` — build the per-cell reference pictures and, optionally, the key-hint
    labels; the three label arguments are all-or-nothing.
  - the three placement overrides and the removal override — each writes through to the
    persistent slot table.
  - `ReloadReferences` — rebuild the whole bar from the slot table against an inventory
    owner. This is the only way the bar's contents change.
  - `LoadItemTexture` — dress one cell from a configuration section's icon rectangle, for
    a slot whose item is not currently carried.
  - `UpdateLabels` — refresh the key hints after a rebinding.
  - `OnItemDBClick` / `OnItemDrop` — unassign, and reorder by swapping slots.
