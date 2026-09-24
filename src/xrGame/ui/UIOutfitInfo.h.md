# src/xrGame/ui/UIOutfitInfo.h

> Declares the protection comparison panel: one row per damage kind, each a label, a
> two-valued bar and a number, stacked and sized to however many rows the layout defines.

**Needs** — [`UIOutfitInfo.cpp`](UIOutfitInfo.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/ProgressBar/UIDoubleProgressBar.h`](../../xrUICore/ProgressBar/UIDoubleProgressBar.h.md) · [`xrServerEntities/alife_space.h`](../../xrServerEntities/alife_space.h.md)
**Used by** — [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) · [`UIItemInfo.cpp`](UIItemInfo.cpp.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`UIOutfitInfo.cpp`](UIOutfitInfo.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIOutfitInfo.cpp`](UIOutfitInfo.cpp.md).

## Exported units

- **`CUIOutfitImmunity`** — one row: a label, a two-valued bar and a value readout, plus the
  scale factor that turns a fraction into whatever the row displays.
- **`CUIOutfitInfo`** — the panel: a row per damage kind that the layout defines, kept stacked.
- `InitFromXml` — build whichever rows the document has.
- `UpdateInfo` in two forms — one for armour, one for helmets — each optionally comparing
  against a second item and, for armour, optionally adding the belt's artefact contribution.
- `AdjustElements` — restack the visible rows and resize the panel to them.
