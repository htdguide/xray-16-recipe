# src/xrUICore/ProgressBar/UIDoubleProgressBar.h

> Declares the comparison bar — two progress bars stacked in one place, showing a current value against a reference and colouring the difference.

**Needs** — [`UIDoubleProgressBar.cpp`](UIDoubleProgressBar.cpp.md) · [`UIProgressBar.h`](UIProgressBar.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md)
**Used by** — [`UIOutfitInfo.cpp`](../../xrGame/ui/UIOutfitInfo.cpp.md) · [`UIOutfitInfo.h`](../../xrGame/ui/UIOutfitInfo.h.md) · [`UIWpnParams.h`](../../xrGame/ui/UIWpnParams.h.md) · [`UIDoubleProgressBar.cpp`](UIDoubleProgressBar.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIDoubleProgressBar.cpp`](UIDoubleProgressBar.cpp.md).

It exists for one screen idiom: showing what a piece of equipment's statistic *would become*
if the player took an action. The longer of the two values is drawn behind in a colour that
says better or worse, and the shorter in front in the ordinary colour, so the visible overhang
is the difference.

## Exported units

- `CUIDoubleProgressBar` — the pair.
- `InitFromXml(document, path)` — builds both bars from the *same* element, then reads two
  extra colours.
- `SetTwoPos(current, reference)` — assign the two values and pick the overhang's colour.

**Notes** — both bars are initialized from one element, so they are identical in geometry,
texture and orientation by construction. The only thing that distinguishes them is which value
each holds and that only the first one draws a backdrop.
