# src/xrUICore/PropertiesBox — the context menu

> A framed list that pops up at the cursor, fits itself inside a bounding rectangle, takes
> mouse and keyboard exclusively while open, and closes on a click outside itself.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The right-click menu: "use", "drop", "unload", "repair". One of these exists per screen that
wants one, is filled with items just before it is shown, and reports the chosen item's
identifier.

## The load-bearing ideas

**Placement is a fit, not a position.** The box is asked to appear near the cursor *inside* a
given rectangle, and it tries placements until one fits — down-right, down-left, up-right,
up-left — so a menu opened near a screen edge folds back rather than being clipped. The same
routine places tooltips.

**Exclusivity is capture plus focus lock, not a flag.** While open it captures the pointer and
locks navigation focus to its own subtree. Nothing else is disabled; everything else simply
stops being reachable. That is the modal mechanism the whole chapter uses.

**Closing on an outside click requires the capture.** Without it the box would never see the
click that should dismiss it, because that click lands on another widget's rectangle.

**The item set is rebuilt per appearance.** A context menu's contents depend on what was
right-clicked, so the box is cleared and refilled each time rather than holding a persistent
list. Its height follows the item count.

## The twins

| Twin | Role |
|---|---|
| [`UIPropertiesBox.cpp`](UIPropertiesBox.cpp.md) | The fit-inside placement, the exclusive capture and focus lock, the outside-click dismissal, and the per-appearance item set |
| [`UIPropertiesBox.h`](UIPropertiesBox.h.md) | Its declaration |
