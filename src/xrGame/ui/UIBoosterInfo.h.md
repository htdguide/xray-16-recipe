# src/xrGame/ui/UIBoosterInfo.h

> Declares the panel that lists what a consumable will do to you, and the one labelled row it
> is built from.

**Needs** — [`UIBoosterInfo.cpp`](UIBoosterInfo.cpp.md) · [`../../xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`UIBoosterInfo.cpp`](UIBoosterInfo.cpp.md) · [`UIItemInfo.cpp`](UIItemInfo.cpp.md) · [`UIItemInfo.h`](UIItemInfo.h.md)
**Tier floor** — T3: reads configuration and stacks rows

## Purpose

Declares the surface implemented in [`UIBoosterInfo.cpp`](UIBoosterInfo.cpp.md).

## `CUIBoosterInfo`

The panel, embedded in the item description. Holds one pre-built row per booster kind plus
three special rows — satiety, surge survival, and effect duration — and shows only the rows
that a given item's configuration actually mentions.

- `InitFromXml(document)` — build every row; returns false when the panel's element is absent,
  which is how a layout omits the panel entirely.
- `SetInfo(section)` — show the rows for one item section and size the panel to fit them.

## `UIBoosterInfoItem`

One row: a caption and a value. Knows how to format its number — a magnitude multiplier, an
optional sign, an optional unit suffix — and can swap its caption icon between a positive and
a negative variant.
