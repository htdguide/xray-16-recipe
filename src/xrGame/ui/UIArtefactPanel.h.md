# src/xrGame/ui/UIArtefactPanel.h

> Declares the overlay strip that shows the artefacts currently on the player's belt.

**Needs** — [`UIArtefactPanel.cpp`](UIArtefactPanel.cpp.md) · [`../../xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`../../xrUICore/Static/UIStaticItem.h`](../../xrUICore/Static/UIStaticItem.h.md)
**Used by** — [`UIArtefactPanel.cpp`](UIArtefactPanel.cpp.md) · [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md)
**Tier floor** — T2: emits quads directly rather than owning child widgets

## Purpose

Declares the surface implemented in [`UIArtefactPanel.cpp`](UIArtefactPanel.cpp.md).

## `CUIArtefactPanel`

A strip of artefact icons drawn on the in-game overlay. Unlike nearly everything else in the
chapter it holds **no child widgets**: it keeps a list of texture rectangles and re-uses a
single drawing primitive for all of them. That is why it can be rebuilt every time the belt
changes without churning the widget tree.

- `InitFromXML(document, path, index)` — geometry, orientation, spacing and scale.
- `InitIcons(artefacts)` — rebuild the rectangle list from the belt's contents.
- `Draw()` — emit one quad per rectangle along the strip.

Laid out horizontally or vertically according to the layout document.
