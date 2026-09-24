# src/xrGame/ui/UIWpnParams.h

> Declares the two comparison panels an item tooltip shows: a weapon's four statistics, and any
> item's condition.

**Needs** — [`UIWpnParams.cpp`](UIWpnParams.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/ProgressBar/UIDoubleProgressBar.h`](../../xrUICore/ProgressBar/UIDoubleProgressBar.h.md)
**Used by** — [`UIItemInfo.cpp`](UIItemInfo.cpp.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`UIWpnParams.cpp`](UIWpnParams.cpp.md)
**Tier floor** — T3: declarations of two panels

## Purpose

Declares the surface implemented in [`UIWpnParams.cpp`](UIWpnParams.cpp.md). Both panels are built
on the **two-position** progress bar, which draws a current value and a comparison value on one
track — the widget that makes "this weapon against the one you are holding" readable at a glance.

Exported units:

- `CUIWpnParams` — the weapon panel: accuracy, damage, handling, rate of fire, plus magazine size
  and ammunition types in single player.
  - `InitFromXml(document)` — build; reports absence so the tooltip can omit the panel.
  - `SetInfo(equipped, hovered)` — fill from the hovered item, comparing against the equipped one.
  - `Check(section)` — whether an item section is a weapon this panel can describe.
- `CUIConditionParams` — the condition panel: one two-position bar and a caption.
