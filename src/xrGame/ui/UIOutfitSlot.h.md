# src/xrGame/ui/UIOutfitSlot.h

> Declares the armour slot: a one-cell drag target that draws a full-length portrait of what is
> worn instead of the item's grid icon.

**Needs** — [`UIOutfitSlot.cpp`](UIOutfitSlot.cpp.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md)
**Used by** — [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIOutfitSlot.cpp`](UIOutfitSlot.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIOutfitSlot.cpp`](UIOutfitSlot.cpp.md).

## Exported units

- **The armour slot** — a drag-and-drop list that draws only a backing picture.
- `SetItem` in three placement forms, and `RemoveItem` — each overridden to refresh the portrait
  after doing what the base does.
- `SetDefaultOutfit` — the portrait to fall back on when nothing is worn and no actor is
  available.
- `Draw` — draws the portrait and nothing else.
